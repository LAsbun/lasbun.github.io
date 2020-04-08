---
title: scrapy(2)之执行分析
date: 2019-07-17 16:46:41
tags: [scrapy, 爬虫]
---

根据之前写的scrapy-1,我们分析到了初始化引擎。下面就实际运行是如何运行的。

<!-- more -->

### scapy.crawler.Crawler.crawl

- 主要工作
    - 实例化spider （这个源码就跳过了，就是简单的创建spider实例，并将参数设置好）
    - 实例化engine (下面详细讲下)
```
    @defer.inlineCallbacks
    def crawl(self, *args, **kwargs):
        assert not self.crawling, "Crawling already taking place"
        self.crawling = True

        try:
            self.spider = self._create_spider(*args, **kwargs)
            ## 这个就是实例化了一个scrapy.core.engine.ExecutionEngine类
            self.engine = self._create_engine()
            # 这里就是从自定义的start_url里面读取url,并封装成 scrapy.http.request.Request实例
            start_requests = iter(self.spider.start_requests())
            yield self.engine.open_spider(self.spider, start_requests)
            yield defer.maybeDeferred(self.engine.start)
        except Exception:
            # In Python 2 reraising an exception after yield discards
            # the original traceback (see https://bugs.python.org/issue7563),
            # so sys.exc_info() workaround is used.
            # This workaround also works in Python 3, but it is not needed,
            # and it is slower, so in Python 3 we use native `raise`.
            if six.PY2:
                exc_info = sys.exc_info()

            self.crawling = False
            if self.engine is not None:
                yield self.engine.close()

            if six.PY2:
                six.reraise(*exc_info)
            raise
    
    def _create_engine(self):
        return ExecutionEngine(self, lambda _: self.stop())
```
这个函数在creat_engine之后，调用了open_spider函数。
### scrapy.core.engine.ExecutionEngine.open_spider
- 主要做了以下动作
    - 将_next_request 注册到twisted
    - 实例化调度器
    - 中间件封装start_request
    - 将事件，start_reqeusts,调度器封装成slot 并将其设置为实例属性
    - 调用scheduler.open函数，初始化去重过滤器中间件（默认是DUPEFILTER_CLASS = 'scrapy.dupefilters.RFPDupeFilter'）
    - 调用scraper.open_spider(spider) 生成pipline
    - slot.nextcall.schedule() 这个是调用前面注册成CallLaterOnce的_next_request函数。

```
@defer.inlineCallbacks
    def open_spider(self, spider, start_requests=(), close_if_idle=True):
        assert self.has_capacity(), "No free spider slot when opening %r" % \
            spider.name
        logger.info("Spider opened", extra={'spider': spider})
        ## 这里是将_next_requests 注册到了twisted
        nextcall = CallLaterOnce(self._next_request, spider)
        scheduler = self.scheduler_cls.from_crawler(self.crawler)
        start_requests = yield self.scraper.spidermw.process_start_requests(start_requests, spider)
        slot = Slot(start_requests, close_if_idle, nextcall, scheduler)
        self.slot = slot
        self.spider = spider
        # 初始化去重过滤器中间件（默认是DUPEFILTER_CLASS = 'scrapy.dupefilters.RFPDupeFilter'）
        yield scheduler.open(spider)
        # 生成pipline
        yield self.scraper.open_spider(spider)
        self.crawler.stats.open_spider(spider)
        yield self.signals.send_catch_log_deferred(signals.spider_opened, spider=spider)
        # 这个是调用前面注册成CallLaterOnce的_next_request函数。
        slot.nextcall.schedule()
        slot.heartbeat.start(5)
```

### _next_reuqests
- 需要注意的是
    - 在调用 _next_request_from_scheduler 的时候，调用了Download
    - 第一次调用_next_request_from_scheduler是没有的数据的，调用self._crawl之后才会吧request放进队列里面
```
    def _next_request(self, spider):
        slot = self.slot
        if not slot:
            return

        if self.paused:
            return
        # 是否需要等待
        while not self._needs_backout(spider):
        # 是否有下一个request，如果没有就break
            if not self._next_request_from_scheduler(spider):
                break
        # 如果还有start_requests 并且没有等待，那么就获取下一个request 并放入队列中
        if slot.start_requests and not self._needs_backout(spider):
            try:
                request = next(slot.start_requests)
            except StopIteration:
                slot.start_requests = None
            except Exception:
                slot.start_requests = None
                logger.error('Error while obtaining start requests',
                             exc_info=True, extra={'spider': spider})
            else:
                # 将request放进队列
                self.crawl(request, spider)

        if self.spider_is_idle(spider) and slot.close_if_idle:
            self._spider_idle(spider)
```
### _next_request_from_scheduler
- 主要做的事情
    - 先从scheduler.next_request() 获取队列中的next_request
    - 调用engine._download 下载（需要注意的是，这里经过了下载中间件） 下面会详细分析
    - 下载完成之后调用engine._handle_downloader_output 后面会讲到。下面先讲下如何下载的。
```
    def _next_request_from_scheduler(self, spider):
        slot = self.slot
        request = slot.scheduler.next_request()
        if not request:
            return
        d = self._download(request, spider)
        ## 
        d.addBoth(self._handle_downloader_output, request, spider)
        d.addErrback(lambda f: logger.info('Error while handling downloader output',
                                           exc_info=failure_to_exc_info(f),
                                           extra={'spider': spider}))
        d.addBoth(lambda _: slot.remove_request(request))
        d.addErrback(lambda f: logger.info('Error while removing request from slot',
                                           exc_info=failure_to_exc_info(f),
                                           extra={'spider': spider}))
        d.addBoth(lambda _: slot.nextcall.schedule())
        d.addErrback(lambda f: logger.info('Error while scheduling new request',
                                           exc_info=failure_to_exc_info(f),
                                           extra={'spider': spider}))
        return d
```

### engine.crawl
- 实际就是把request放进调度器重
```
    def crawl(self, request, spider):
        assert spider in self.open_spiders, \
            "Spider %r not opened when crawling: %s" % (spider.name, request)
        self.schedule(request, spider)
        self.slot.nextcall.schedule()

    def schedule(self, request, spider):
        self.signals.send_catch_log(signal=signals.request_scheduled,
                request=request, spider=spider)
        if not self.slot.scheduler.enqueue_request(request):
            self.signals.send_catch_log(signal=signals.request_dropped,
                                        request=request, spider=spider)
```

### scheduler.enqueue_request
- 如果计算指纹 参考看下scrapy.utils.request.py中的request_fingerprint函数
```
    def enqueue_request(self, request):
    # 如果不需要过滤 或者是指纹重复就不进队列
        if not request.dont_filter and self.df.request_seen(request):
            self.df.log(request, self.spider)
            return False
        dqok = self._dqpush(request)
        if dqok:
            self.stats.inc_value('scheduler/enqueued/disk', spider=self.spider)
        else:
            self._mqpush(request)
            self.stats.inc_value('scheduler/enqueued/memory', spider=self.spider)
        self.stats.inc_value('scheduler/enqueued', spider=self.spider)
        return True
```

上面介绍了如果request如何进队列，下面介绍下scrapy是如何下载request

### ExecutionEngine._download

```
    def _download(self, request, spider):
        slot = self.slot
        slot.add_request(request)
        def _on_success(response):
            assert isinstance(response, (Response, Request))
            if isinstance(response, Response):
                response.request = request # tie request to response received
                logkws = self.logformatter.crawled(request, response, spider)
                logger.log(*logformatter_adapter(logkws), extra={'spider': spider})
                self.signals.send_catch_log(signal=signals.response_received, \
                    response=response, request=request, spider=spider)
            return response

        def _on_complete(_):
            slot.nextcall.schedule()
            return _
        # 这里下载 调用的是scrapy.core.download.Downloader.fetch
        dwld = self.downloader.fetch(request, spider)
        # 注册成功之后的调用
        dwld.addCallbacks(_on_success)
        # 注册完成
        dwld.addBoth(_on_complete)
        return dwld
```

### scrapy.core.download.Downloader.fetch
- 注意
    - 这里有个时间是完成之后，会从active中去除掉改request
```
    def fetch(self, request, spider):
        def _deactivate(response):
            self.active.remove(request)
            return response

        self.active.add(request)
        ## 这里调用的是scrapy.core.downloader.DownloaderMiddlewareManager
        dfd = self.middleware.download(self._enqueue_request, request, spider)
        return dfd.addBoth(_deactivate)
```
### scrapy.core.downloader.DownloaderMiddlewareManager.download
- 注意
    - 这个函数主要是调用了下载中间件，最后再proecss_request执行完之后调用** scrapy.core.downloader.Downloader._enqueue_request** 才会真正的调用的下载
    - 这里注册了三个事件, 这些调用定义的下载中间件。 默认的请参考default_setting.py  DOWNLOADER_MIDDLEWARES_BASE配置
        - process_request  处理下载
        - process_response 处理返回
        - process_exception 处理异常
    - process_request 处理完之后会调用self._enqueue_request函数  这个函数
```
    def download(self, download_func, request, spider):
        @defer.inlineCallbacks
        def process_request(request):
            for method in self.methods['process_request']:
                response = yield method(request=request, spider=spider)
                if response is not None and not isinstance(response, (Response, Request)):
                    raise _InvalidOutput('Middleware %s.process_request must return None, Response or Request, got %s' % \
                                         (six.get_method_self(method).__class__.__name__, response.__class__.__name__))
                if response:
                    defer.returnValue(response)
            defer.returnValue((yield download_func(request=request, spider=spider)))

        @defer.inlineCallbacks
        def process_response(response):
            assert response is not None, 'Received None in process_response'
            if isinstance(response, Request):
                defer.returnValue(response)

            for method in self.methods['process_response']:
                response = yield method(request=request, response=response, spider=spider)
                if not isinstance(response, (Response, Request)):
                    raise _InvalidOutput('Middleware %s.process_response must return Response or Request, got %s' % \
                                         (six.get_method_self(method).__class__.__name__, type(response)))
                if isinstance(response, Request):
                    defer.returnValue(response)
            defer.returnValue(response)

        @defer.inlineCallbacks
        def process_exception(_failure):
            exception = _failure.value
            for method in self.methods['process_exception']:
                response = yield method(request=request, exception=exception, spider=spider)
                if response is not None and not isinstance(response, (Response, Request)):
                    raise _InvalidOutput('Middleware %s.process_exception must return None, Response or Request, got %s' % \
                                         (six.get_method_self(method).__class__.__name__, type(response)))
                if response:
                    defer.returnValue(response)
            defer.returnValue(_failure)
        ## 注册事件
        deferred = mustbe_deferred(process_request, request)
        deferred.addErrback(process_exception)
        deferred.addCallback(process_response)
        return deferred
```
### scrapy.core.downloader.Downloader._enqueue_request/_process_queue/_download
- 这里涉及到了三个Downloader的函数
    -_enqueue_request 将request 和spider封装成slot 放进队列后，调用_process_queue
    - _process_queue 在判断是否需要延迟之后，调用_download，进行下载
    - _download 调用self.handlers.download_request(默认的是scrapy.core.downloader.handlers.DownloadHandlers)，这个函数会根据不同的scheme选择不同的handler. 默认的分为
        - ``` DOWNLOAD_HANDLERS_BASE = {
            'data': 'scrapy.core.downloader.handlers.datauri.DataURIDownloadHandler',
            'file': 'scrapy.core.downloader.handlers.file.FileDownloadHandler',
            'http': 'scrapy.core.downloader.handlers.http.HTTPDownloadHandler',
            'https': 'scrapy.core.downloader.handlers.http.HTTPDownloadHandler',
            's3': 'scrapy.core.downloader.handlers.s3.S3DownloadHandler',
            'ftp': 'scrapy.core.downloader.handlers.ftp.FTPDownloadHandler',
        }
          ```

```
    def _enqueue_request(self, request, spider):
        key, slot = self._get_slot(request, spider)
        request.meta[self.DOWNLOAD_SLOT] = key

        def _deactivate(response):
            slot.active.remove(request)
            return response

        slot.active.add(request)
        self.signals.send_catch_log(signal=signals.request_reached_downloader,
                                    request=request,
                                    spider=spider)
        deferred = defer.Deferred().addBoth(_deactivate)
        slot.queue.append((request, deferred))
        self._process_queue(spider, slot)
        return deferred

    def _process_queue(self, spider, slot):
        if slot.latercall and slot.latercall.active():
            return

        # Delay queue processing if a download_delay is configured
        now = time()
        delay = slot.download_delay()
        if delay:
            penalty = delay - now + slot.lastseen
            if penalty > 0:
                slot.latercall = reactor.callLater(penalty, self._process_queue, spider, slot)
                return

        # Process enqueued requests if there are free slots to transfer for this slot
        while slot.queue and slot.free_transfer_slots() > 0:
            slot.lastseen = now
            request, deferred = slot.queue.popleft()
            dfd = self._download(slot, request, spider)
            dfd.chainDeferred(deferred)
            # prevent burst if inter-request delays were configured
            if delay:
                self._process_queue(spider, slot)
                break

    def _download(self, slot, request, spider):
        # The order is very important for the following deferreds. Do not change!

        # 1. Create the download deferred
        dfd = mustbe_deferred(self.handlers.download_request, request, spider)

        # 2. Notify response_downloaded listeners about the recent download
        # before querying queue for next request
        def _downloaded(response):
            self.signals.send_catch_log(signal=signals.response_downloaded,
                                        response=response,
                                        request=request,
                                        spider=spider)
            return response
        dfd.addCallback(_downloaded)

        # 3. After response arrives,  remove the request from transferring
        # state to free up the transferring slot so it can be used by the
        # following requests (perhaps those which came from the downloader
        # middleware itself)
        slot.transferring.add(request)

        def finish_transferring(_):
            slot.transferring.remove(request)
            self._process_queue(spider, slot)
            return _

        return dfd.addBoth(finish_transferring)
```
至此，下载以及下载中间件的处理已经完成了。我们现在回到### _next_request_from_scheduler,看下前面所讲到的engine._handle_downloader_output。

### ExecutionEngine._handle_downloader_output
- 如果返回的response是request 那么久调用crawl方法，将request放入scheduler
- 如果返回的response 不是request。调用scraper.enqueue_scrape方法，处理response.
```
    def _handle_downloader_output(self, response, request, spider):
        assert isinstance(response, (Request, Response, Failure)), response
        # downloader middleware can return requests (for example, redirects)
        if isinstance(response, Request):
            self.crawl(response, spider)
            return
        # response is a Response or Failure
        d = self.scraper.enqueue_scrape(response, request, spider)
        d.addErrback(lambda f: logger.error('Error while enqueuing downloader output',
                                            exc_info=failure_to_exc_info(f),
                                            extra={'spider': spider}))
        return d
```
### scrapy.core.scraper.Scraper.enqueue_scrape
- 注册结束处理事件(从active队列中移除掉)
```
    # 将结果放入slot队列中
    def enqueue_scrape(self, response, request, spider):
        slot = self.slot
        dfd = slot.add_response_request(response, request)
        # 这个就是处理完成之后，已出队列中response
        def finish_scraping(_):
            slot.finish_response(response, request)
            self._check_if_closing(spider, slot)
            self._scrape_next(spider, slot)
            return _
        dfd.addBoth(finish_scraping)
        dfd.addErrback(
            lambda f: logger.error('Scraper bug processing %(request)s',
                                   {'request': request},
                                   exc_info=failure_to_exc_info(f),
                                   extra={'spider': spider}))
        self._scrape_next(spider, slot)
        return dfd
    
    # 取出队列中的元素，进行处理
    def _scrape_next(self, spider, slot):
        while slot.queue:
            response, request, deferred = slot.next_response_request_deferred()
            self._scrape(response, request, spider).chainDeferred(deferred)
    # 注册 两个时间
    def _scrape(self, response, request, spider):
        """Handle the downloaded response or failure through the spider
        callback/errback"""
        assert isinstance(response, (Response, Failure))

        dfd = self._scrape2(response, request, spider) # returns spiders processed output
        # 这个是处理错误
        dfd.addErrback(self.handle_spider_error, request, response, spider)
        # 处理callback
        dfd.addCallback(self.handle_spider_output, request, response, spider)
        return dfd

    # 如果成功就调用下载中间件以及在经过中间件处理完之后调用call_spider函数，调用，否则就记录下载失败相关信息
    def _scrape2(self, request_result, request, spider):
        """Handle the different cases of request's result been a Response or a
        Failure"""
        ## 这里调用的是scrapy.core.spidermw.SpiderMiddlewareManager.scrape_response
        if not isinstance(request_result, Failure):
            return self.spidermw.scrape_response(
                self.call_spider, request_result, request, spider)
        else:
            dfd = self.call_spider(request_result, request, spider)
            return dfd.addErrback(
                self._log_download_errors, request_result, request, spider)

    # 调用spider.callback和spider.parse函数
    def call_spider(self, result, request, spider):
        result.request = request
        dfd = defer_result(result)
        dfd.addCallbacks(request.callback or spider.parse, request.errback)
        return dfd.addCallback(iterate_spider_output)

```
### scrapy.core.spidermw.SpiderMiddlewareManager.scrape_response
- 先调用process_spider_input, 预处理response 注意此时没有调用spider中的callback或者是parse函数
- 执行完process_spider_input后 调用scrapy.core.scraper.Scraper.call_spider函数，
- 如果有异常会调用process_spider_exception
- 如果无异常会调用 process_spider_output 
```
    def scrape_response(self, scrape_func, response, request, spider):
        fname = lambda f:'%s.%s' % (
                six.get_method_self(f).__class__.__name__,
                six.get_method_function(f).__name__)

        def process_spider_input(response):
            for method in self.methods['process_spider_input']:
                try:
                    result = method(response=response, spider=spider)
                    if result is not None:
                        raise _InvalidOutput('Middleware {} must return None or raise an exception, got {}' \
                                             .format(fname(method), type(result)))
                except _InvalidOutput:
                    raise
                except Exception:
                    return scrape_func(Failure(), request, spider)
            return scrape_func(response, request, spider)

        def process_spider_exception(_failure, start_index=0):
            exception = _failure.value
            # don't handle _InvalidOutput exception
            if isinstance(exception, _InvalidOutput):
                return _failure
            method_list = islice(self.methods['process_spider_exception'], start_index, None)
            for method_index, method in enumerate(method_list, start=start_index):
                if method is None:
                    continue
                result = method(response=response, exception=exception, spider=spider)
                if _isiterable(result):
                    # stop exception handling by handing control over to the
                    # process_spider_output chain if an iterable has been returned
                    return process_spider_output(result, method_index+1)
                elif result is None:
                    continue
                else:
                    raise _InvalidOutput('Middleware {} must return None or an iterable, got {}' \
                                         .format(fname(method), type(result)))
            return _failure

        def process_spider_output(result, start_index=0):
            # items in this iterable do not need to go through the process_spider_output
            # chain, they went through it already from the process_spider_exception method
            recovered = MutableChain()

            def evaluate_iterable(iterable, index):
                try:
                    for r in iterable:
                        yield r
                except Exception as ex:
                    exception_result = process_spider_exception(Failure(ex), index+1)
                    if isinstance(exception_result, Failure):
                        raise
                    recovered.extend(exception_result)

            method_list = islice(self.methods['process_spider_output'], start_index, None)
            for method_index, method in enumerate(method_list, start=start_index):
                if method is None:
                    continue
                # the following might fail directly if the output value is not a generator
                try:
                    result = method(response=response, result=result, spider=spider)
                except Exception as ex:
                    exception_result = process_spider_exception(Failure(ex), method_index+1)
                    if isinstance(exception_result, Failure):
                        raise
                    return exception_result
                if _isiterable(result):
                    result = evaluate_iterable(result, method_index)
                else:
                    raise _InvalidOutput('Middleware {} must return an iterable, got {}' \
                                         .format(fname(method), type(result)))

            return chain(result, recovered)

        dfd = mustbe_deferred(process_spider_input, response)
        ## 注册了这个callback事件
        dfd.addCallbacks(callback=process_spider_output, errback=process_spider_exception)
        return dfd
```

### 结
至此，scrapy crawl xxx 整体流程分析完毕

### 核心类图

- 其中褐色表示方法
- 淡黄表示属性
- 仔细观察还是结合scrapy整体架构图的

{% asset_img image-20190725153242445.png  核心类图 %}

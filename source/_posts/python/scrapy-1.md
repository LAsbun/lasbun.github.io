---
title: scrapy(1)深入-环境配置
date: 2019-07-02 16:40:59
tags: [scrapy, 爬虫]
---
scrapy 是python 爬虫框架中用度最广，且简单的流行款。这里笔者依据源码，分析下 
 **scrapy crawl xxx** 之后的操作。

<!-- more -->
    
###  scrapy crawl xxx 调用流程

- 由setup.py 
```
entry_points={
        'console_scripts': ['scrapy = scrapy.cmdline:execute']
    }
```
我们找到开始的python 脚本 scrapy.cmdline:execute
- 主要做了下面几件事情
    - 初始化环境配置，在项目中运行项目
    - 初始化CrawlProcess
    - 调用run
    ```
    def execute(argv=None, settings=None):
        if argv is None:
            argv = sys.argv

        # --- 兼容之前版本配置 ---
        if settings is None and 'scrapy.conf' in sys.modules:
            from scrapy import conf
            if hasattr(conf, 'settings'):
                settings = conf.settings
        # ------------------------------------------------------------------

        # 初始化scrapy.cfg 以及 scrapy.settings.default_setting.py 中的配置
        if settings is None:
            settings = get_project_settings()
            # set EDITOR from environment if available
            try:
                editor = os.environ['EDITOR']
            except KeyError: pass
            else:
                settings['EDITOR'] = editor
        # 输出deprecated 配置
        check_deprecated_settings(settings)

        # --- 兼容之前版本配置---
        import warnings
        from scrapy.exceptions import ScrapyDeprecationWarning
        with warnings.catch_warnings():
            warnings.simplefilter("ignore", ScrapyDeprecationWarning)
            from scrapy import conf
            conf.settings = settings
        # ------------------------------------------------------------------
        
        # 判断是否在project中
        inproject = inside_project()
        # 加载支持的命令 即package scrapy.commands下面的所有的文件 集合成字典
        cmds = _get_commands_dict(settings, inproject)

        cmdname = _pop_command_name(argv)
        parser = optparse.OptionParser(formatter=optparse.TitledHelpFormatter(), \
            conflict_handler='resolve')
        if not cmdname:
            _print_commands(settings, inproject)
            sys.exit(0)
        elif cmdname not in cmds:
            _print_unknown_command(settings, cmdname, inproject)
            sys.exit(2)

        cmd = cmds[cmdname]
        parser.usage = "scrapy %s %s" % (cmdname, cmd.syntax())
        parser.description = cmd.long_desc()
        settings.setdict(cmd.default_settings, priority='command')
        cmd.settings = settings
        cmd.add_options(parser)
        # 解析参数
        opts, args = parser.parse_args(args=argv[1:])
        # 这里是对命令中一些参数进行设置 比如是 ```-logfile=xxx.xx```等
        _run_print_help(parser, cmd.process_options, args, opts)
        # 爬虫运行类
        cmd.crawler_process = CrawlerProcess(settings)
        # 执行 cmd 中的run类。 ** 在这里就是执行scrapy.commands.crawler.run函数** 
        _run_print_help(parser, _run_command, cmd, args, opts)
        sys.exit(cmd.exitcode)
    ```
### scray.crawler.CrawlerProcess 

- 这个类是继承了scray.crawler.CrawlerRunner，主要是增加了一个start函数。
需要注意的是__init__ 函数。
- scray.crawler.CrawlerRunner.__init__ 中有个属性是spider_loader 是所有爬虫类的加载器，默认使用的是 default_setting.py中的 SPIDER_LOADER_CLASS ='scrapy.spiderloader.SpiderLoader'。

### scrapy.commands.crawler.run

- 这里就是调用了CrawlProcess.crawl/start 函数

    ```
    def run(self, args, opts):
        if len(args) < 1:
            raise UsageError()
        elif len(args) > 1:
            raise UsageError("running 'scrapy crawl' with more than one spider is no longer supported")
        spname = args[0]
        
        self.crawler_process.crawl(spname, **opts.spargs)
        self.crawler_process.start()

        if self.crawler_process.bootstrap_failed:
            self.exitcode = 1
    ```

### CrawlProcess.crawl/CrawlerRunner.crawl

- 实际上这个调用的是父类 **rawlerRunner.crawl**
- 主要工作
    - 初始化并返回一个scrapy.crawler.Crawler实例
    - 调用_crawl函数
    ```
        def crawl(self, crawler_or_spidercls, *args, **kwargs):
            
            if isinstance(crawler_or_spidercls, Spider):
                raise ValueError(
                    'The crawler_or_spidercls argument cannot be a spider object, '
                    'it must be a spider class (or a Crawler object)')
            ## 返回一个scrapy.crawler.Crawler对象
            crawler = self.create_crawler(crawler_or_spidercls)
            ## 调用 CrawlerRunner._crawl 函数
            return self._crawl(crawler, *args, **kwargs)
    ```

### scrapy.crawler.Crawler 初始化
    ```
    def __init__(self, spidercls, settings=None):
        if isinstance(spidercls, Spider):
            raise ValueError(
                'The spidercls argument must be a class, not an object')

        if isinstance(settings, dict) or settings is None:
            settings = Settings(settings)

        self.spidercls = spidercls
        self.settings = settings.copy()
        self.spidercls.update_settings(self.settings)

        d = dict(overridden_settings(self.settings))
        logger.info("Overridden settings: %(settings)r", {'settings': d})

        self.signals = SignalManager(self)
        self.stats = load_object(self.settings['STATS_CLASS'])(self)

        handler = LogCounterHandler(self, level=self.settings.get('LOG_LEVEL'))
        logging.root.addHandler(handler)
        if get_scrapy_root_handler() is not None:
            # scrapy root handler already installed: update it with new settings
            install_scrapy_root_handler(self.settings)
        # lambda is assigned to Crawler attribute because this way it is not
        # garbage collected after leaving __init__ scope
        self.__remove_handler = lambda: logging.root.removeHandler(handler)
        self.signals.connect(self.__remove_handler, signals.engine_stopped)

        lf_cls = load_object(self.settings['LOG_FORMATTER'])
        self.logformatter = lf_cls.from_crawler(self)
        self.extensions = ExtensionManager.from_crawler(self)

        self.settings.freeze()
        self.crawling = False
        # spider 实例 初始化
        self.spider = None
        # 核心engine
        self.engine = None
    ```

### CrawlerRunner._crawl

- 主要工作
    - 上面初始化了一个Crawler 对象， 调用Crawler.crawl函数（初始化引擎等其他配置）
    
```
    def _crawl(self, crawler, *args, **kwargs):
        self.crawlers.add(crawler)
        // Crawler.crawl
        d = crawler.crawl(*args, **kwargs)
        self._active.add(d)

        def _done(result):
            self.crawlers.discard(crawler)
            self._active.discard(d)
            self.bootstrap_failed |= not getattr(crawler, 'spider', None)
            return result

        return d.addBoth(_done)

```

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
            ## 这个就是实例化了一个scrapy.engine.ExecutionEngine类
            self.engine = self._create_engine()
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

### scrapy.engine.ExecutionEngine

- __init__ 函数主要工作
    - 加载调度器(默认是SCHEDULER = 'scrapy.core.scheduler.Scheduler')
    - 加载下载器(默认是DOWNLOADER = 'scrapy.core.downloader.Downloader')
    - 初始化 scrapy.core.scrapyer.Scraper 对象(下面详解)
    ```
    class ExecutionEngine(object):

        def __init__(self, crawler, spider_closed_callback):
            self.crawler = crawler
            self.settings = crawler.settings
            self.signals = crawler.signals
            self.logformatter = crawler.logformatter
            self.slot = None
            self.spider = None
            self.running = False
            self.paused = False
            self.scheduler_cls = load_object(self.settings['SCHEDULER'])
            downloader_cls = load_object(self.settings['DOWNLOADER'])
            self.downloader = downloader_cls(crawler)
            # 
            self.scraper = Scraper(crawler)
            self._spider_closed_callback = spider_closed_callback
    ```

### scrapy.core.scrapyer.Scraper

- 主要工作 (连接  middleware pipline spider)
    - 初始化中间件
    - 初始化item
```
class Scraper(object):

    def __init__(self, crawler):
        self.slot = None
        self.spidermw = SpiderMiddlewareManager.from_crawler(crawler)
        itemproc_cls = load_object(crawler.settings['ITEM_PROCESSOR'])
        self.itemproc = itemproc_cls.from_crawler(crawler)
        self.concurrent_items = crawler.settings.getint('CONCURRENT_ITEMS')
        self.crawler = crawler
        self.signals = crawler.signals
        self.logformatter = crawler.logformatter
```
到此核心插件初始化就完成了。
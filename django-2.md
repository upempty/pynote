## django main entry
```
python manager.py runserver ...
===============================>

execute_from_command_line()
    utility.execute()
		--->django.setup()
	self.fetch_command('runserver').run_from_argv(self.argv)
		--->fetch_command('runserver') 返回对应命令模块的类的实例，这里是'runserver'命令的类
		--->run_from_argv(self.argv)
			--->execute()做一些设置参数的错误检查，然后设置句柄----in BaseCommand, then-->self.handle(*args, **options).
				--->handle()做些设置---in Command(BaseCommand), also execute, but calling super.execute
					--->run() 主要调用inner_run，区分是否是use_reloader
						--->inner_run()
								--->inner_run() 设置句柄handler,WSGIHandler 的实例
									---run(ip,port,handler...,WSGIServer)一个标准的 wsgi 实现, here handler like application(wsgi_application)
                                                                           it's in basehttp.py, one function not in class but function.
                                                                              ->httpd=httpd_cls(addr, WSGIRequestHandler,,,)
                                                                              ->httpd.set_app(handler.....also application)
                                                                              ->httpd.serve_forever-------------socketserver.py there, there are self._handle_request_noblock()...
```

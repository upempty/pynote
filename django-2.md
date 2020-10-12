## django main entry
```
python manager.py runserver ...
===============================>
main()
->execute_from_command_line(sys.argv)
    -->utility.execute()--ManagementUtility(argv).execute()
	-->django.setup()
	-->self.fetch_command('runserver').run_from_argv(self.argv)
		--->fetch_command('runserver') 'runserver' Command class
		  --->run_from_argv(self.argv)
			--->execute()----in BaseCommand, then-->self.handle(*args, **options).
				--->handle()---in Command(BaseCommand), also execute, but calling super.execute
					--->run() check if use_reloader
						--->inner_run()
						     --->run(ip,port,handler...,WSGIServer)here handler like application(wsgi_application)
                                                             it's in basehttp.py, one function not in class but function.
                                                           ->httpd=httpd_cls(addr, WSGIRequestHandler,,,)
                                                           ->httpd.set_app(handler.....also application)
                                                           ->httpd.serve_forever---insocketserver.py there, there are self._handle_request_noblock()
```
## django handle request 
```
reference: https://blog.csdn.net/bingjia103126/article/details/105466669/

```

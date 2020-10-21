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
                                                           ->httpd.serve_forever---in socketserver.py there, there are self._handle_request_noblock()
---------
    BaseServer._handle_request_noblock()

        BaseServer.get_request() -> request, client_addres

        BaseServer.verify_request()

            BaseServer.process_request()

                BaseServer.process_request()

                    BaseServer.finish_request()

                        BaseServer.RequestHandlerClass()

                            BaseRequestHandler.__init__(request)
            
                                BaseRequestHandler.request
                                BaseRequestHandler.client_address = client_address

                                BaseRequestHandler.setup()

                                BaseRequestHandler.handle()------>class WSGIRequestHandler(simple_server.WSGIRequestHandler).handle()->.handle_one_request->
				   ->handler = ServerHandler(...)--->handler.run(self.server.get_app())
---------

BaseHandler:
    def run(self, application):
        """Invoke the application"""
        # Note to self: don't move the close()!  Asynchronous servers shouldn't
        # call close() from finish_response(), so if you close() anywhere but
        # the double-error branch here, you'll break asynchronous servers by
        # prematurely closing.  Async servers must return from 'run()' without
        # closing if there might still be output to iterate over.
        try:
            self.setup_environ()
            self.result = application(self.environ, self.start_response)--------------
            self.finish_response()---------xx

def _get_response(self, request)
    response = wrapped_callback(request, *callback_args, **callback_kwargs) -------------to self.result


    def finish_response(self):-------------xx
        """Send any iterable data, then close self and the iterable

        Subclasses intended for use in asynchronous servers will
        want to redefine this method, such that it sets up callbacks
        in the event loop to iterate over the data, and to call
        'self.close()' once the response is finished.
        """
        try:
            if not self.result_is_file() or not self.sendfile():
                for data in self.result:----------------------------------------------
                    self.write(data)------------------------------------------------------
		    
    def write(self, data):
        """'write()' callable as specified by PEP 3333"""

        assert type(data) is bytes, \
            "write() argument must be a bytes instance"

        if not self.status:
            raise AssertionError("write() before start_response()")

        elif not self.headers_sent:
            # Before the first output, send the stored headers
            self.bytes_sent = len(data)    # make sure we know content-length
            self.send_headers()----------------------------------------------------------------send self.headers firstly
        else:
            self.bytes_sent += len(data)

        # XXX check Content-Length and truncate if too many bytes written?
        self._write(data)----------------------------------------------------------------------xxx! send data content like HttpResponse() in views.py


class FirstView(View):
    def get(self, request, *args, **kwargs):
        result = {'status': True, 'data': 'first response'}
        print('test=', result)
        return JsonResponse(result, status = 200)----------------------------------------------xxx!
	
	
	




```
## django handle request at running time
```
reference: https://blog.csdn.net/bingjia103126/article/details/105466669/

```

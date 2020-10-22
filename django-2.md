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
                                                     --->handler = self.get_handler(*args, **options)->get_internal_wsgi_application->
						            app_path = getattr(settings, 'WSGI_APPLICATION')
                                                            if app_path is None:
                                                                return get_wsgi_application()
						            try:
                                                                return import_string(app_path)---blog.wsgi.application
								   ---from django.core.wsgi import get_wsgi_application-->WSGIHandler()(envrion, start_response)
								       during init of WSGIHandler() it is calling load_middleware()
								   ------- request = self.request_class(environ)
                                                                           response = self.get_response(request)--->response = self._middleware_chain(request)!!
									      下面的self._get_response最终会赋给_middleware_chain！！！所以->_middle...->_get_res...()
									   ///-->
_middleware_chain(request)->here _middleware_chain ==== below									   
        
	
	///
	    def _get_response(self, request):
        """
        Resolve and call the view, then apply view, exception, and
        template_response middleware. This method is everything that happens
        inside the request/response middleware.
        """
        response = None
        callback, callback_args, callback_kwargs = self.resolve_request(request)----------'get' 'post' like....！！！！！！！！！！！！！
	  -->        # Resolve the view, and assign the match object back to the request.
                     resolver_match = resolver.resolve(request.path_info); request.resolver_match = resolver_match; return resolver_match
		     
        wrapped_callback = self.make_view_atomic(callback)
	response = wrapped_callback(request, *callback_args, **callback_kwargs)

        -----more including middleware view code refer as below code.
        # Apply view middleware
        for middleware_method in self._view_middleware:
            response = middleware_method(request, callback, callback_args, callback_kwargs)
            if response:
                break

        if response is None:
            wrapped_callback = self.make_view_atomic(callback)
            # If it is an asynchronous view, run it in a subthread.
            if asyncio.iscoroutinefunction(wrapped_callback):
                wrapped_callback = async_to_sync(wrapped_callback)
            try:
                response = wrapped_callback(request, *callback_args, **callback_kwargs)
		
	///
	calling load_middleware---during init: asigned to _middleware_chain.
	 get_response = self._get_response_async if is_async else self._get_response
         handler = convert_exception_to_response(get_response)
	
	 for middleare path in xxx:
	      if hasattr(mw_instance, 'process_view'):
                self._view_middleware.insert(
                    0,
                    self.adapt_method_mode(is_async, mw_instance.process_view),)
		    
	  handler = self.adapt_method_mode(
                    middleware_is_async, handler, handler_is_async,
                    debug=settings.DEBUG, name='middleware %s' % middleware_path,
                )
	  mw_instance = middleware(handler)
	
	  handler = convert_exception_to_response(mw_instance) 
	
	handler = self.adapt_method_mode(is_async, handler, handler_is_async)
        # We only assign to this when initialization is complete as it is used
        # as a flag for initialization being complete.
        self._middleware_chain = handler---------------------------
	
	
									   ///-->
									   
									   
									   
									   
									   .....
									   start_response(status, response_headers)---self.headers assigned here.
						     
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
        return JsonResponse(result, status = 200)----------------------------------------------xxx! JsonResponse() object's items is values or content, 
	---------------------------------------------------------------------------------------so to iteral to list, for x in result.
	
	
	




```
## django handle request at running time
```
reference: https://blog.csdn.net/bingjia103126/article/details/105466669/

```

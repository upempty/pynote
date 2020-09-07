```
pip install requests==2.23.0 -i https://pypi.doubanio.com/simple/

installation timeout issue resolution. To use native country mirroring to speed up the installation.

pip3 install --upgrade pip -i https://pypi.doubanio.com/simple/

or --default-timeout=100
pip3 install virtualenv --index https://pypi.mirrors.ustc.edu.cn/simple/
virtualenv -p python3 venv3
source bin/activate
pip list
pip3 install django -i https://pypi.doubanio.com/simple/
pip freeze



➜  ~ python3
Python 3.7.2 (v3.7.2:9a3ffc0492, Dec 24 2018, 02:59:38)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import django
>>>

VS code
control + - ; go back
  cmd + click ; go definition
  
debug->add config
lauch.json
port 9000
path
fn+f5 for execute

/Users/feicheng/dev/py/django3/django3-course/.vscode
(venv3) ➜  .vscode git:(master) ✗ cat launch.json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [


        {
            "name": "Python: Django",
            "type": "python",
            "request": "launch",
            "pythonPath": "/Users/feicheng/dev/py/venv3/bin/python",
            "program": "/Users/feicheng/dev/py/django3/django3-course/02.Django是如何工作的/manage.py",
            "args": [
                "runserver",
                "9000",
                "--noreload"
            ],
            "django": true
        }
    ]
}%


(venv3) ➜  django3-blog git:(master) python manage.py createsuperuser
Username (leave blank to use 'feicheng'): test
Email address: 184918308@qq.com
Password:
Password (again):
The password is too similar to the username.
This password is too short. It must contain at least 8 characters.
This password is too common.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
(venv3) ➜  django3-blog git:(master)

To use "run without debug" in "Debug" Menu if object_list is not variable issue.
```

## start_response usage, seems it is not important?
```
--cpython/Lib/wsgiref/handlers.py:
class BaseHandler:
    """Manage the invocation of a WSGI application"""

    # Configuration parameters; can override per-subclass or per-instance
    wsgi_version = (1,0)
    wsgi_multithread = True
    wsgi_multiprocess = True
    wsgi_run_once = False

    origin_server = True    # We are transmitting direct to client
    http_version  = "1.0"   # Version that should be used for response
    server_software = None  # String name of server software, if any

    # os_environ is used to supply configuration from the OS environment:
    # by default it's a copy of 'os.environ' as of import time, but you can
    # override this in e.g. your __init__ method.
    os_environ= read_environ()

    # Collaborator classes
    wsgi_file_wrapper = FileWrapper     # set to None to disable
    headers_class = Headers             # must be a Headers-like class

    # Error handling (also per-subclass or per-instance)
    traceback_limit = None  # Print entire traceback to self.get_stderr()
    error_status = "500 Internal Server Error"
    error_headers = [('Content-Type','text/plain')]
    error_body = b"A server error occurred.  Please contact the administrator."

    # State variables (don't mess with these)
    status = result = None
    headers_sent = False
    headers = None
    bytes_sent = 0

    def run(self, application):
        """Invoke the application"""
        # Note to self: don't move the close()!  Asynchronous servers shouldn't
        # call close() from finish_response(), so if you close() anywhere but
        # the double-error branch here, you'll break asynchronous servers by
        # prematurely closing.  Async servers must return from 'run()' without
        # closing if there might still be output to iterate over.
        try:
            self.setup_environ()
            self.result = application(self.environ, self.start_response)
            
--django/django/core/handlers/wsgi.py:           
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.load_middleware()

    def __call__(self, environ, start_response):
        set_script_prefix(get_script_name(environ))
        signals.request_started.send(sender=self.__class__, environ=environ)
        request = self.request_class(environ)
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = [
            *response.items(),
            *(('Set-Cookie', c.output(header='')) for c in response.cookies.values()),
        ]
        start_response(status, response_headers)
        
        
--cpython/Lib/wsgiref/handlers.py:

 def start_response(self, status, headers,exc_info=None):
        """'start_response()' callable as specified by PEP 3333"""

        if exc_info:
            try:
                if self.headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[0](exc_info[1]).with_traceback(exc_info[2])
            finally:
                exc_info = None        # avoid dangling circular ref
        elif self.headers is not None:
            raise AssertionError("Headers already set!")

        self.status = status
        self.headers = self.headers_class(headers)
        status = self._convert_string_type(status, "Status")
        assert len(status)>=4,"Status must be at least 4 characters"
        assert status[:3].isdigit(), "Status message must begin w/3-digit code"
        assert status[3]==" ", "Status message must have a space after code"

        if __debug__:
            for name, val in headers:
                name = self._convert_string_type(name, "Header name")
                val = self._convert_string_type(val, "Header value")
                assert not is_hop_by_hop(name),\
                       f"Hop-by-hop header, '{name}: {val}', not allowed"

        return self.write
```

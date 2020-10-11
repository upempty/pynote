## env for python path for search source code usage
```
VSCODE-Code->Preferences->Settings search: Python Path
/Users/feicheng/dev/py/venv3/bin/python
```

## verify
http://127.0.0.1:8000/firstV/
```
result====> {"status": true, "data": "first response"}
```

## code 
blog/urls.py
```
urlpatterns = [
    path('admin/', admin.site.urls),
    path("firstV/", include("rest.urls")),
]
``` 

rest/urls.py
```
from django.urls import path
from rest.views import FirstView

urlpatterns = [
    path('', FirstView.as_view()),
]
```

rest/views.py
```
from django.shortcuts import render
from django.views import View 
from django.http import JsonResponse
# Create your views here.

class FirstView(View):
    def get(self, request, *args, **kwargs):
        result = {'status': True, 'data': 'first response'}
        print('test=', result)
        return JsonResponse(result, status = 200)
```

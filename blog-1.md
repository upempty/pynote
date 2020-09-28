## ceate django project/app
```
pip install djangorestframework -i https://pypi.doubanio.com/simple/

(venv3) ➜  plan1 django-admin.py startproject blog
(venv3) ➜  blog ls
blog      manage.py
(venv3) ➜  blog ls blog
__init__.py asgi.py     settings.py urls.py     wsgi.py

settings.py:
ALLOWED_HOSTS = ['*']

python manage.py startapp rest
(venv3) ➜  blog ls
blog       db.sqlite3 manage.py  rest
(venv3) ➜  blog cd rest
(venv3) ➜  rest ls
__init__.py apps.py     models.py   views.py

Debug->Add Configuration

{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Django",
            "type": "python",
            "pythonPath": "/Users/feicheng/dev/py/venv3/bin/python",
            "request": "launch",
            "program": "${workspaceFolder}/blog/manage.py",
            "args": [
                "runserver",
                "--noreload"
            ],
            "django": true
        }
    ]
}
// /Users/feicheng/dev/upempty/plan1/.vscode/launch.json

```

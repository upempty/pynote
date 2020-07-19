'''
# installation timeout issue resolution. To use native country mirroring to speed up the installation.

pip3 install --upgrade pip -i https://pypi.doubanio.com/simple/



# or --default-timeout=100
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

# VS code
# control + - ; go back
  cmd + click ; go definition
  

'''

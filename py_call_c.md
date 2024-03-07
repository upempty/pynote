```
==mygod.c

#include <stdio.h>
int layer_comm_add (void);
int layer_comm_add (void)
{
  printf("Now be %s \n", "communicated");
  return 0;
}

gcc -c -fPIC mygod.c
gcc -shared -o libmygod.so *.o
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH


===pmygod.py
#!/usr/bin/python
import ctypes
from ctypes import *
lmygod = ctypes.CDLL("//home/f1cheng/lastcode/libmygod.so")

print('begin')
ret = lmygod.layer_comm_add()
print('end with ret={}'.format(ret))
#recommend for above version python3.5 
print(f'end with ret={ret}')

==printout of python3 pmygod.py
begin
Now be communicated
end with ret=0
end with ret=0
```

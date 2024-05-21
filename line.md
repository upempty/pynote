- __line__ print

```
cat test21.c

#include <stdio.h>


#define STATE(State0, State1)  \
     State0 = __LINE__;  \
     State1 = __LINE__;


int main ()
{
    printf ("This is line %d of file \"%s\".\n",
            __LINE__, __FILE__);
    int s0, s1;
    STATE(s0, s1); // line 15
    printf ("s0=%d, s1=%d\n", s0, s1);
    return 0;
}

/*
 *
 *$./a.out
  This is line 13 of file "test21.c".
  s0=15, s1=15
 *
 *
 */


```

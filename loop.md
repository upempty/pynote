
- dead loop without block operation will occupy CPU a lot, but it will be still scheduled out and in.
```
Depends on:
1- time slices used out, will switch to another task.
2- if this loop has high priority it will continue occupy CPU.
3- if other task yield() or sleep(), then it has chance to re-occupy.
4- if task amount is less, this loop task may get more CPU time.
```

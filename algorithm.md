
- 最小公倍数
```
def minG(x, y):
    ret = max(x, y)
    smaller = min(x, y)
    i = 0
    for i in range(1, smaller+1):
        mod = (ret * i )%smaller
        if mod == 0:
            ret = ret * i
            break
    return ret

p = minG(5, 10)
print(f'result={p}')

```

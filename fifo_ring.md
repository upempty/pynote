
- 	kfifo
  - https://github.com/torvalds/linux/blob/master/lib/kfifo.c#L89

```
fifo->in += len;
fifo->in +=  len而未取模，unsigned int的溢出又会被置为0.

```
  - https://github.com/torvalds/linux/blob/2c8159388952f530bd260e097293ccc0209240be/include/linux/kfifo.h#L4
```
struct __kfifo {
	unsigned int	in;
	unsigned int	out;
	unsigned int	mask;
	unsigned int	esize;
	void		*data;
};
```
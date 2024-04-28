
- kfifo
    - https://github.com/torvalds/linux/blob/master/lib/kfifo.c#L89

```
fifo->in += len;
fifo->in +=  len未取模是ok的，因为unsigned int的溢出又会被置为0.

fifo->in % fifo->size => when size=2的幂次方  fifo->in & (fifo->size – 1)

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
- 无锁实现
  - https://www.cnblogs.com/sniperHW/p/4172248.html
  - https://www.cnblogs.com/zengzy/p/5145899.html
  - 
```
  typedef struct _Node Node;
    typedef struct _Queue Queue;

    struct _Node {
        void *data;
        Node *next;
    };

    struct _Queue {
        Node *head;
        Node *tail;
    };

    Queue*
    queue_new(void)
    {
        Queue *q = g_slice_new(sizeof(Queue));
        q->head = q->tail = g_slice_new0(sizeof(Node));
        return q;
    }

    void
    queue_enqueue(Queue *q, gpointer data)
    {
        Node *node, *tail, *next;

        node = g_slice_new(Node);
        node->data = data;
        node->next = NULL;

        while (TRUE) {
            tail = q->tail;
            next = tail->next;
            if (tail != q->tail)
                continue;

            if (next != NULL) {
                CAS(&q->tail, tail, next);
                continue;
            }

            if (CAS(&tail->next, null, node)
                break;
        }

        CAS(&q->tail, tail, node);
    }

    gpointer
    queue_dequeue(Queue *q)
    {
        Node *node, *tail, *next;

        while (TRUE) {
            head = q->head;
            tail = q->tail;
            next = head->next;
            if (head != q->head)
                continue;

            if (next == NULL)
                return NULL; // Empty

            if (head == tail) {
                CAS(&q->tail, tail, next);
                continue;
            }

            data = next->data;
            if (CAS(&q->head, head, next))
                break;
        }

        g_slice_free(Node, head); // This isn't safe
        return data;
    }       
```  
   

```
template<typename ELEM_TYPE>
 2 class queue{
 3 public:
 4     queue(int size);//把size上调至2的幂保存到que_size中
 5     bool enqueue(const ELEM_TYPE& in_data);
 6     bool dequeue(ELEM_TYPE& out_data );
 7     ...//其他成员
 8 private:
 9     ELEM_TYPE *arry;
10     int in;
11     int out;
12     int max_out;//用于读写线程同步
13     int que_size;
14     ... //其他成员
15 }
16 
17 template <typename ELEM_T>
18 bool queue<ELEM_T>::enqueue(const ELEM_T& in_data)
19 {
20     int cur_in ;
21     int cur_out;
22 
23     do{
24         cur_in  = in;
25         cur_out = out;
26 
27         if(cur_in - cur_out == que_size){
28             return false;
29         }
30         
31     }while(!CAS(&in,cur_in,cur_in+1))//如果cur_in==in依然成立，说明没有其他线程修改in，cur_in是可用的，并修改in;如果cur_in!=in,说明in值已被其他线程修改
32     
33     arry[cur_in&(que_size-1)] = in_data;
34     
35     while(!CAS(&max_out,cur_in,cur_in+1)){//发布数据
36         sched_yield();//发布数据失败，cur_in之前还有数据没有发布出去,让出cpu,让其他线程先执行
37     }
38     
39     return true;
40 }
41 
42 template <typename ELEM_T>
43 bool queue<ELEM_T,QUE_SIZE>::dequeue(ELEM_T& out_data)
44 {
45     int cur_out;
46     int cur_max_out;
47 
48     do{
49         cur_out   = out;
50         cur_max_out = max_out;
51 
52         if(cur_out == cur_max_out){
53             return false;
54         }
55 
56         out_data = arry[cur_out&(que_size-1)] ;//先尝试获取数据
57         
58     }while(!CAS(&out,cur_out,cur_out+1))
59 
60     return true;
61 }
```

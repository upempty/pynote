```
python pcom.py
```

```
//layer_communication.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/shm.h>
#include <string.h>
#include <pthread.h>

#define MAX_BUFFERS 10
#define CONTAINER_KEY 102
#define QUEUE_KEY 103

typedef struct Containers
{
    
    char buf[MAX_BUFFERS][256];
    int buffer_index;
    int reserved[MAX_BUFFERS];
    pthread_mutex_t lock; // not used.
} Containers;

typedef struct MsgQueues
{
    
    int msgPos[256];
    int head;
    int size; //(head+size)%len(sizeof(msgIDs)), size++; head = (head+1)%(sizeof(msgIDs)), size--
    pthread_mutex_t lock; // not used.
} MsgQueues;

int initContainers(void)
{
    Containers *containers;
    int key = CONTAINER_KEY;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int container_head_len = 4;
    shmid = shmget((key_t)key, sizeof(Containers) + container_head_len, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);
    containers = (struct Containers*)((void *)shm_addr + container_head_len);
    int *length = (int *)shm_addr;
    *length = 0;
    memset(containers, 0, sizeof(containers));
    printf("write to shared memory=%d\n", key);
    return 0;
}

int layer_mem_reserve(int len, char *str)
{
    printf("layer_mem_reserve:=%s \n", str);

    Containers *containers;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int container_head_len = 4;
    shmid = shmget((key_t)CONTAINER_KEY, sizeof(Containers) + container_head_len, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);
    containers = (struct Containers*)((void *)shm_addr + container_head_len);

    printf("write to shared memory=%d\n", CONTAINER_KEY);
    
    //char *str = "first msg to sent";
    int msg_len = strlen(str) + 1;
    containers->buffer_index = 1;
    //to find free slots index
    int index = 0;
    containers->reserved[index] = msg_len;
    strcpy(containers->buf[index], str);
    printf("buf index=%d \n", containers->buffer_index);
    printf("buf content=%s \n", containers->buf[index]);
    //to use sem_t for lock/unlok, sem_open, sem_wait, sem_post
    return index;
}

char* layer_mem_receive(int cid)
{
    printf("layer_mem_release");
    Containers *containers;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int container_head_len = 4;
    shmid = shmget((key_t)CONTAINER_KEY, sizeof(Containers) + container_head_len, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);
    containers = (struct Containers*)((void *)shm_addr + container_head_len);

    printf("write to shared memory=%d\n", CONTAINER_KEY);
    
    //char *str = "first msg to sent";
    containers->buffer_index = 1;
    //to find slot
    // static or other way!!!
    //static char tmpbuf [256];
    //memcpy(tmpbuf, containers->buf[cid], containers->reserved[cid]);
    //!!containers->reserved[cid] = 0;
    printf("buf index=%d \n", containers->buffer_index);
    printf("buf content=%s \n", containers->buf[cid]);

    //to use sem_t for lock/unlok, sem_open, sem_wait, sem_post
    return containers->buf[cid];
}


int layer_mem_release(int cid)
{
    printf("layer_mem_release");
    Containers *containers;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int container_head_len = 4;
    shmid = shmget((key_t)CONTAINER_KEY, sizeof(Containers) + container_head_len, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);
    containers = (struct Containers*)((void *)shm_addr + container_head_len);

    printf("write to shared memory=%d\n", CONTAINER_KEY);
    
    //char *str = "first msg to sent";
    containers->buffer_index = 1;
    //to find slot
    // static or other way!!!
    //static char tmpbuf [256];
    //memcpy(tmpbuf, containers->buf[cid], containers->reserved[cid]);
    containers->reserved[cid] = 0; //!!! important
    printf("buf index=%d \n", containers->buffer_index);
    printf("buf content=%s \n", containers->buf[cid]);

    //to use sem_t for lock/unlok, sem_open, sem_wait, sem_post
    return 0;
}


int initQueues(void) 
{
    //SmsgLocation localqueue[128];

    MsgQueues *msgQ;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int hh = 4;
    shmid = shmget((key_t)QUEUE_KEY, sizeof(MsgQueues) + hh, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);
    msgQ = (struct MsgQueues*)((void *)shm_addr + hh);
    int *length = (int *)shm_addr;
    *length = 0;
    memset(msgQ, 0, sizeof(MsgQueues));
    printf("write to shared memory=%d\n", 1);
    return 0;
}

int layer_push_queue(int qid, int cid)
{
    printf("layer_push_queue");
    MsgQueues *msgQ;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int hh = 4;
    shmid = shmget((key_t)QUEUE_KEY, sizeof(MsgQueues) + hh, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);

    msgQ = (struct MsgQueues*)((void *)shm_addr + hh);
    int sof_msgQ = sizeof(msgQ->msgPos)/sizeof(int);
    printf("write to shared memory=%d\n", sof_msgQ);
    int pos = (msgQ->head+msgQ->size) % sizeof(msgQ->msgPos)/sizeof(int);
    msgQ->msgPos[pos] = cid;
    msgQ->size++;
    printf("shared memory updated head=%d, size=%d\n", msgQ->msgPos[pos], msgQ->size);
    return 0;
}

int layer_pop_queue(int qid)
{
    printf("layer_pop_queue");
    MsgQueues *msgQ;
    void *shm_addr;
    char tmp[20];
    int shmid;
    int hh = 4;
    shmid = shmget((key_t)QUEUE_KEY, sizeof(MsgQueues) + hh, 0666 | IPC_CREAT);
    printf("Key of shared memory is %d\n", shmid);
    shm_addr = shmat(shmid, NULL, 0);
    printf("Process attached at %p\n", shm_addr);
    msgQ = (struct MsgQueues*)((void *)shm_addr + hh);

    int cid = msgQ->msgPos[msgQ->head];
    msgQ->head = (msgQ->head + 1) % sizeof(msgQ->msgPos)/sizeof(int);
    msgQ->size--;
    printf("shared memory updated head=%d, size=%d\n", msgQ->head, msgQ->size);//head = (head+1)%(sizeof(msgIDs)), size--

    return cid;
}



/*
gcc -c -fPIC layer_communication.c -o 
gcc -shared -o lib_layercomm.so layer_communication.o
// issue gcc -shared -o lib_layercomm.so layer_communication.c

int main()
{

    initContainers();
    initQueues();

    layer_mem_reserve(20, "are you ok");
    layer_push_queue(0, 0);

    int cid = layer_pop_queue(0);
    char* msg = layer_mem_receive(0);
    printf("------ss=notyetrelease msg=%s--------------", msg);
    layer_mem_release(0);

}
*/
```


```
#pcom.py
import ctypes
from ctypes import *
libcom = ctypes.CDLL("//myhome/programming/lib_layercomm.so")
initContainers = libcom.initContainers
initQueues = libcom.initQueues

layer_mem_reserve = libcom.layer_mem_reserve
layer_mem_reserve.argtypes = [ctypes.c_int, ctypes.c_char_p]
layer_mem_reserve.restype = ctypes.c_int

layer_push_queue = libcom.layer_push_queue
layer_push_queue.argtypes = [ctypes.c_int, ctypes.c_int]
layer_push_queue.restype = ctypes.c_int

layer_pop_queue = libcom.layer_pop_queue
layer_pop_queue.argtypes = [ctypes.c_int]
layer_pop_queue.restype = ctypes.c_int

layer_mem_receive = libcom.layer_mem_receive
layer_mem_receive.argtypes = [ctypes.c_int]
layer_mem_receive.restype = ctypes.c_char_p

layer_mem_release = libcom.layer_mem_release
layer_mem_release.argtypes = [ctypes.c_int]
layer_mem_release.restype = ctypes.c_int

ret = initContainers()
ret = initQueues()
str = "are you ok in py"

print('testing in ctypes\n')
ret = layer_mem_reserve(ctypes.c_int(20), str.encode('utf-8'))
print(f'py1: ret = {ret} layer_mem_reserve')
ret = layer_push_queue(ctypes.c_int(0),ctypes.c_int(0))
print(f'py1: ret = {ret} layer_push_queue')
ret = layer_pop_queue(ctypes.c_int(0))
print(f'py1: ret = {ret} layer_pop_queue')
retstr = layer_mem_receive(ctypes.c_int(0))
print(f'py1: ret = {retstr} layer_mem_receive')
retstr2 = ctypes.string_at(retstr).decode('utf-8')
print(f'py1: ret = {retstr2} layer_mem_receive')
ret = layer_mem_release(0)
print(f'py1: ret = {ret}  layer_mem_release')


'''
    initContainers();
    initQueues();


    layer_mem_reserve(20, "are you ok");
    layer_push_queue(0, 0);

    int cid = layer_pop_queue(0);
    char* msg = layer_mem_receive(0);
    printf("------ss=notyetrelease msg=%s--------------", msg);
    layer_mem_release(0);
'''
```



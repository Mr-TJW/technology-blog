# 数据队列
[TOC]
---
在日常收发数据过程中，尤其多线程操作，数据的收发需要用到数据队列去处理，那么为什么要使用数据队列？什么是数据队列呢？

## 常规存储机制弊端

在接收或者发送数据的时候，你的数据的存储机制是什么样的呢 ？
* 是否是采用下述方式？

```c
#define MAX_BUF_SIZE    2048

unsigned short  recv_len;
unsigned char   recv_buf[MAX_BUF_SIZE];

int main()
{
    ......
    while(1)
    {
        recv_len = receive(recv_buf);
        ......
    }
}
```

* 此种方法如果在单线程中问题不大，但是不适合多线程，多线程中，通常接收和发送函数是和主线程分离开来的，例如：

```c

#define MAX_BUF_SIZE    2048

unsigned short  recv_len;
unsigned char   recv_buf[MAX_BUF_SIZE];

void *recv_thread(void *arg)
{   
    int ret;

    while(1)
    {   
        ret = receive(recv_buf); /* 如果接收到数据，则返回值为接收字节数 */
        if(ret > 0)
        {
            recv_len = ret;
        }
    }
}

int main()
{   
    pthread_t thread_other;
    ......
    
    /* 创建接收线程，主要作用是负责接收数据 */
    pthread_create (&thread_other, NULL, recv_thread, NULL);

    while(1)
    {
        if(recv_len > 0)
        {
            /* 处理接收 buf */
            ......
        }
    }
    return 0;
}

```

**代码弊端**：上述代码模仿单线程的接收处理，但是显然问题很大，因为处理和接收在不同线程，处理线程正在处理数据过程中，有可能接收线程又接收到新数据，导致处理线程处理的数据被覆盖掉，会导致意想不到的后果。

**解决方法**：多线程操作中，我们希望：如果前一次接收到的数据还没有处理结束，此时又有新数据到来，我们希望新数据不会将未处理的数据覆盖掉，而是存储到未处理数据后面，等上一次数据处理结束后再处理新数据，这就是**数据队列**的存储机制。

## 数据队列的实现

数据队列的核心就是数据的入栈和出栈。下述代码中，`Data_Queue` 就是数据队列，`PutData` 和 `DropData` 分别是数据的入栈和出栈：

```c

#define MAX_DATA_SIZE   2048

typedef struct  
{
    unsigned short start;               //目前处理到的数据索引
    unsigned short next;                //最新入栈的数据索引
    unsigned short count;               //待处理数据数量
    unsigned short size;                //缓冲区大小
    unsigned char  buf[MAX_DATA_SIZE];  //数据缓冲区
}Data_Queue;

/*******************************************************************
* funcname:	PutData
* para:	    data_queue-数据队列，data-入栈数据
* function:	数据入栈
* return:	
********************************************************************/
void PutData(Data_Queue *data_queue, unsigned char data)
{
    (*data_queue).buf[(*data_queue).next++] = data;         //数据入栈

    if((*data_queue).next >= (*data_queue).size)            //入栈数据索引超过到缓冲区最大索引后，归零
    {
        (*data_queue).next = 0;
    }

    (*data_queue).count++;
    if((*data_queue).count >= (*data_queue).size)           //未处理的数据超多缓冲区容量后，舍弃旧数据，保留新数据
    {
        (*data_queue).count = (*data_queue).size;
        (*data_queue).start++;
        if((*data_queue).start >= (*data_queue).size)       //已处理数据索引超过到缓冲区最大索引后，归零
        {
            (*data_queue).start = 0;
        }
    }
}

/*******************************************************************
* funcname:	DropData
* para:	    data_queue-数据队列
* function:	数据出栈
* return:	
********************************************************************/
void DropData(Data_Queue *data_queue)
{
    if((*data_queue).next == (*data_queue).start)       //数据处理完毕直接返回
    {
        (*data_queue).count = 0;
        return;
    }

    (*data_queue).start++;                              //已处理数据索引 + 1
    if((*data_queue).start >= (*data_queue).size)
    {
        (*data_queue).start = 0;
    }

    if((*data_queue).count > 0)                         //未处理数据 -1
    {
        (*data_queue).count--;
    }
    
    return;
    

}
```

## 数据队列实战
下面使用数据队列方式，解决常规存储机制的弊端：
```c

#define MAX_BUF_SIZE    2048

typedef struct  
{
    unsigned short start;               //目前处理到的数据索引
    unsigned short next;                //最新入栈的数据索引
    unsigned short count;               //待处理数据数量
    unsigned short size;                //缓冲区大小
    unsigned char  buf[MAX_BUF_SIZE];   //数据缓冲区
}Data_Queue;

Data_Queue data_queue;

void *recv_thread(void *arg)
{   
    int ret,i;
    unsigned short  recv_len;
    unsigned char   recv_buf[MAX_BUF_SIZE];

    while(1)
    {   
        recv_len = receive(recv_buf); /* 如果接收到数据，则返回值为接收字节数 */
        if(recv_len > 0)
        {
            for(i=0;i<recv_len;i++)
            {
                PutData(&data_queue,recv_buf[i]);
            }
        }
    }
}

int read(unsigned char *read_buf)
{   
    int i = 0;
    while(data_queue.count)
    {
        read_buf[i++] = data_queue.buf[start];
        DropData(&data_queue);
    }

    return i;
}

int main()
{   
    unsigned char read_buf[MAX_BUF_SIZE];
    pthread_t thread_other;
    ......
    
    /* 创建接收线程，主要作用是负责接收数据 */
    pthread_create (&thread_other, NULL, recv_thread, NULL);

    while(1)
    {
        if(read(read_buf) > 0)
        {
            /* 处理接收 buf */
            ......
        }
    }
    return 0;
}


```

## 总结
上述主要以接收为例，说明了使用数据队列的原因以及用法，发送数据其实也一样，只需要在发送前 PutData，然后在发送结束后，DropData 即可。
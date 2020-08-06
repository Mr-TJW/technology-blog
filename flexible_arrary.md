# 柔型数组详解
------------------------

## 柔型数组的概念
> 结构中的最后一个元素允许是未知大小的数组，这就叫做柔性数组(flexible array)成员(也叫伸缩性数组成员)。

在日常编程中，有时需要在结构体中存放一个长度是动态的字符串(也可能是其他数据类型)，一般的做法，实在结构体中定义一个指针成员，这个指针成员指向该字符串所在的动态内存空间。这种方法的最大缺点是导致结构体成员地址不连续，导致内存碎片化。在通常情况下，**如果想要高效的利用内存，那么在结构体内部定义静态的数组是非常浪费的行为**。其实柔性数组的想法和动态数组的想法是一样的。

## 不使用柔性数组案例
```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define BUF_LEN     5

typedef struct 
{   
    char    len; 
    char    *para_buf;
}para_struct;

int main(void)
{
    int i=0;
    para_struct *para;

    para = (para_struct *)malloc(sizeof(para_struct)+BUF_LEN*sizeof(char));  
    printf("sizeof(char) = %d\r\n",sizeof(char));
    printf("sizeof(char*) = %d\r\n",sizeof(char*));
    printf("sizeof(para_struct) = %d\r\n",sizeof(para_struct));
    printf("para struct addr = %p\r\n",para);
    printf("para len    addr = %p\r\n",&para->len);
    printf("para buf1   addr = %p\r\n",para->para_buf);

    para->para_buf = (char *)(para+1);  /* 将 wait_malloc_para 指针指向后面申请的 256 字节空间 */
    memset((char *)(para+1),0x55,BUF_LEN);
    printf("para buf2   addr = %p\r\n",para->para_buf);

    for(i=0;i<BUF_LEN;i++)
    printf("para buf[%d] = %#x\r\n",i,para->para_buf[i]);
        
    free(para);
    return 0;
}
```
运行结果：
![](![](2020-08-06-14-37-50.png).png)

运行结果分析：
* 结构体 `para_struct` 中 len 占1字节，para_buf 指针占4字节，但 `sizeof(para_struct) = 8`，因为结构体的字节对齐所导致。
* malloc 分配的第一段 para_struct 空间中的 para_buf 此时的 para_buf 还是个野指针，并没有指向任何数据。
* `para->para_buf = (char *)(para+1);` 中 `para+1 ` 的意义是让其跳过 mallloc 分配的第一段para_struct 地址，让 para_buf 直接指向后面为其开辟的 BUF_LEN 字节空间，即para_buf 也需要另外分配指向
* 可以看出，这种方法得到的 para_buf 和 len 其实地址并不是连续的，增加了内存的碎片化。

## 使用柔性数组案例

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define BUF_LEN     5

typedef struct 
{   
    char    len; 
    char    para_buf[0];
}para_struct;

int main(void)
{
    int i=0;
    para_struct *para;

    para = (para_struct *)malloc(sizeof(para_struct)+BUF_LEN*sizeof(char));  
    printf("sizeof(para_struct) = %d\r\n",sizeof(para_struct));
    printf("para struct addr = %p\r\n",para);
    printf("para len    addr = %p\r\n",&para->len);
    printf("para buf    addr = %p\r\n",para->para_buf);
    memset(para->para_buf,0x55,BUF_LEN);

    for(i=0;i<BUF_LEN;i++)
    printf("para buf[%d] = %#x\r\n",i,para->para_buf[i]);
        
    free(para);
    return 0;
}
```
运行结果：
![](2020-08-06-15-06-35.png)

运行结果分析:
* para_struct 结构体仅占 1 个字节，因数组下标为 0，不占空间。
* 且 para_buf 不需要额外分配空间，结构体成员地址连续。

## 使用注意事项
* 结构中的柔性数组成员前面必须至少一个其他成员。
* 结构体中最后一个数组长度为零，有些编译器会报错无法编译，可以改成char data[1],或则char data[]。 
* 总之，结构体最后使用0或1的长度数组的原因，主要是为了方便的管理内存缓冲区，当使用指针时候（使用方法一时），不能分配一段连续的的内存，会增加内存的碎片化。  
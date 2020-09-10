# offsetof 结构体元素偏移量计算
> C 库宏 offsetof(type, member-designator) 会生成一个类型为 size_t 的整型常量，可以计算一个结构成员相对于结构开头的字节偏移量。

## 描述
**offsetof** 是一个宏，在 stddef.h 中定义，可以计算一个结构成员相对于结构开头的字节偏移量。

## 宏的定义

`#define offsetof(struct_t,member) ((size_t)(char *)&((struct_t *)0)->member)`

* (struct_t *)0 ：是一个指向struct_t类型的指针，其指针值为 0，所以其作用就是把从地址 0 开始的存储空间映射为一个 struct_t 类型的对象。

* ((struct_t *)0)->member ：是访问类型中的成员 member，相应地 &((struct_t *)0)->member) 就是返回这个成员的地址。

由于对象的起始地址为 0，所以成员的地址其实就是相对于对象首地址的成员的偏移地址。然后在通过类型转换，转换为 size_t 类型（size_t一般是无符号整数）。

所以，offsetoff(struct_t,member)宏的作用就是获得成员 member 在类型 struct_t 中的偏移量。

## 宏的声明

`offsetof(struct_t,member)`

参数：
* type ： 结构类型    
* member ： 结构成员

返回值：
该宏返回类型为 size_t 的值，表示 type 中成员的偏移量。

## 示例

```c
#include <stddef.h>
#include <stdio.h>

struct address {
   char name[50];
   char street[50];
   int phone;
};
   
int main()
{
    printf("address 结构中的 name 偏移 = %d 字节。\n",
    offsetof(struct address, name));

    printf("address 结构中的 street 偏移 = %d 字节。\n",
    offsetof(struct address, street));

    printf("address 结构中的 phone 偏移 = %d 字节。\n",
    offsetof(struct address, phone));

    return(0);
} 
```

运行结果：
```c
address 结构中的 name 偏移 = 0 字节。
address 结构中的 street 偏移 = 50 字节。
address 结构中的 phone 偏移 = 100 字节。
```
# 快速排序详解
------------------------
> 快速排序是 C.R.A.Hoare 于 1962 年提出的一种划分交换排序。它采用了一种分治法的策略，通常称其为分治法 (Divide-and-ConquerMethod)。

## 基本思想

* 1．先从数列中取出一个数作为基准数。
* 2．分区过程，将比这个数大的数全放到它的右边，小于或等于它的数全放到它的左边。
* 3．再对左右区间重复第二步，直到各区间只有一个数。

虽然快速排序采用了分治法的策略，但是分治法一词很难概括快速排序的的思想，因此概括快速排序更更倾向于使用<font color=red> 填坑法 + 分治法 </font>去概括。

示例：
加入有一组数据，如下所示：
|index|0|1|2|3|4|5|6|7|8|9|
|---|---|---|---|---|---|---|---|---|---|---|
|arrary|75|12|53|82|34|68|42|88|52|71|

i 为自左向右的索引值，j 为自右向左额索引值，初始时，i = 0,j = 9。

* 1.选取索引 0 上的数据为基准值


```c
// #include "stdio.h"
// #include "stdlib.h"

// typedef struct tag_Basic_Data {
//     int value;
//     int index;
// }BascData;

// int findBasic(int left,int right,int arrary[])
// {
//     int temp,i,j;
//     int rightFlag=1;
//     int basic;

//     i = left,j = right;
    
//     basic = arrary[left];

//     while(i < j)
//     {
//         if(rightFlag == 1)
//         {
//             if(arrary[j] < basic )
//             {
//                 arrary[i++] = arrary[j];
//                 rightFlag = 0;
//             }
//             else
//             {
//                 j--;
//             }
            
//         }
//         else
//         {
//             if(arrary[i] > basic )
//             {
//                 arrary[j--] = arrary[i];
//                 rightFlag = 1;
//             }
//             else
//             {
//                 i++;
//             }
            
//         }
//     }

//     printf("basic = %d\n",basic);
//     arrary[i] = basic;
//     return i;
// }

// void quickSort(int left,int right,int arrary[])
// {
//     int ret;
//     if(left < right)
//     {   
//         ret = findBasic(left,right,arrary);

//         quickSort(left,ret-1,arrary);
//         quickSort(ret+1,right,arrary);
//     }

// }

// int main()
// {
//     int i,j;
//     int arrary[] = {56,85,42,22,16,78,20,69,85,12,75,63,18};

//     printf("primary data:");
//     for(i=0;i<sizeof(arrary)/sizeof(arrary[0]);i++)
//     {
//         printf("%d ",arrary[i]);
//     }
//     printf("\n");

//     quickSort(0,sizeof(arrary)/sizeof(int)-1,arrary);

//     printf("after sort data:");
//     for(i=0;i<sizeof(arrary)/sizeof(arrary[0]);i++)
//     {
//         printf("%d ",arrary[i]);
//     }
//     printf("\n");

//     getchar();
//     return;
// }

#include "stdio.h"
#include "stdlib.h"

typedef struct tag_Basic_Data {
    int value;
    int index;
}BascData;

int findBasic(int left,int right,int arrary[])
{
    
}

void quickSort(int left,int right,int arrary[])
{
    int ret;
    int i,j;
    int rightFlag=1;
    int basic;
    
    if(left < right) {
        i = left,j = right;
        basic = arrary[left];
        while(i < j)
        {
            if(rightFlag == 1) {
                if(arrary[j] < basic ) {
                    arrary[i++] = arrary[j];
                    rightFlag = 0;
                }
                else j--;
            }
            else {
                if(arrary[i] > basic ) {
                    arrary[j--] = arrary[i];
                    rightFlag = 1;
                }
                else i++;
            }
        }
        arrary[i] = basic;
        quickSort(left,i-1,arrary);
        quickSort(i+1,right,arrary);
    }
}

int main()
{
    int i,j;
    int arrary[] = {56,85,42,22,16,78,20,69,85,12,75,63,18,1,5,3,5453,43,633,223,553,22,143,66};

    printf("primary data:");
    for(i=0;i<sizeof(arrary)/sizeof(arrary[0]);i++)
    {
        printf("%d ",arrary[i]);
    }
    printf("\n");

    quickSort(0,sizeof(arrary)/sizeof(int)-1,arrary);

    printf("after sort data:");
    for(i=0;i<sizeof(arrary)/sizeof(arrary[0]);i++)
    {
        printf("%d ",arrary[i]);
    }
    printf("\n");

    getchar();
    return;
}


```
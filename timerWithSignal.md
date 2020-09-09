# 使用信号实现定时器

在 Linux 系统下，每一个进程都有惟一的一个定时器，该定时器提供了以秒为单位的定时功能。在定时器设置的超时时间到达后，调用 alarm 的进程将收到 SIGALRM 信号。

## alarm 函数

* 函数原型：

```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
```

* 参数说明：
seconds：要设定的定时时间，以秒为单位。在 alarm 调用成功后開始计时。超过该时间将触发 SIGALRM 信号。

* 返回值：
返回当前进程曾经设置的定时器剩余秒数。

## 示例

```c
#include <stdio.h>
#include <signal.h>


int Cnt=0;  //全局计数器变量

/* SIGALRM 信号处理函数 */
void CbSigAlrm(int signo)
{
    
    printf("seconds: %d\n",++Cnt);    //输出定时提示信息
    alarm(1);   //又一次启动定时器，实现1秒定时
}

int main()
{
    if(signal(SIGALRM,CbSigAlrm) == SIG_ERR)  //安装SIGALRM信号
    {
        perror("signal");
        return;
    }

    setbuf(stdout,NULL);    //关闭标准输出的行缓存模式
    
    alarm(1);               //启动定时器
    
    while(1)                //进程进入无限循环，仅仅能手动终止
    {   
        pause();            //暂停，等待信号
    }
}
```
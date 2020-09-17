# Linux C 学习之路

## waitpid 函数

* 作用
会暂时停止目前进程的执行，直到有信号来到或子进程结束

* 头文件
```c
#include<sys/types.h>
#include<sys/wait.h>
```

* 原型
pid_t waitpid(pid_t pid,int *status,int options);

* 函数说明：
    * 如果在调用 waitpid() 时子进程已经结束,则 waitpid()会立即返回子进程结束状态值。 
    * 子进程的结束状态值会由参数 status 返回,而子进程的进程识别码也会一起返回。如果不在意结束状态值,则参数 status 可以设成 NULL。参数 pid 为欲等待的子进程识别码,

* 参数
    * pid
        * pid=-1 等待任何子进程,相当于 wait()。
        * pid=0 等待进程组识别码与目前进程相同的任何子进程。
        * pid>0 等待任何子进程识别码为 pid 的子进程。
    * options 提供了一些额外的选项来控制 waitpid,可以为 0 或可以用 "|" 运算符把它们连接起来使用。
        * WNOHANG 若pid指定的子进程没有结束，则 waitpid() 函数返回0，不予以等待。若结束，则返回该子进程的ID。
        * WUNTRACED 若子进程进入暂停状态，则马上返回，但子进程的结束状态不予以理会。WIFSTOPPED(status) 宏确定返回值是否对应与一个暂停子进程。

* 示例
  
```c
/******
* waitpid.c - Simple wait usage
*********/
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>
int main( void )
{
    pid_t childpid;
    int status;
    childpid = fork();
    if ( childpid < 0 )
    {
        perror( "fork()" );
        exit( EXIT_FAILURE );
    }
    else if ( childpid == 0 )
    {
        puts( "In child process" );
        sleep( 3 );//让子进程睡眠3秒，看看父进程的行为
        printf("\tchild pid = %d\n", getpid());
        printf("\tchild ppid = %d\n", getppid());
        exit(EXIT_SUCCESS);
    }
    else
    {
        waitpid( childpid, &status, 0 );
        puts( "in parent" );
        printf( "\tparent pid = %d\n", getpid() );
        printf( "\tparent ppid = %d\n", getppid() );
        printf( "\tchild process exited with status %d \n", status );
    }
    exit(EXIT_SUCCESS);
}
```
运行结果：
```c
[root@localhost src]# gcc waitpid.c
[root@localhost src]# ./a.out
In child process
child pid = 4469
child ppid = 4468
in parent
parent pid = 4468
parent ppid = 4379
child process exited with status 0
[root@localhost src]#
```

如果将 `waitpid( childpid, &status, 0 );` 注释掉，执行效果如下：
```c
[root@localhost src]# ./a.out
In child process
in parent
parent pid = 4481
parent ppid = 4379
child process exited with status1
[root@localhost src]# child pid = 4482
child ppid = 1
```
子进程还没有退出，父进程已经退出了。

## fork 和 vfork 的区别

1. fork()：子进程拷贝父进程的数据段，代码段，vfork()：子进程与父进程共享数据段 
2. fork() :父子进程的执行次序不确定，vfork 保证子进程先运行，在调用exec 或exit 之前与父进程数据是共享的,在它调用exec或exit 之后父进程才可能被调度运行。 
3. vfork() 保证子进程先运行，在她调用exec 或exit 之后父进程才可能被调度运行。如果在调用这两个函数之前子进程依赖于父进程的进一步动作，则会导致死锁。 

## scanf("%*s") 的含义

* 作用：
`scanf("%*s")` 表示跳至下一空白字符，这里主要是中间的 '*' 字符起的作用。

* 示例1
```c
    int n;
    scanf("%*d %*d %d",&n);
    printf("%d",n);
    return 0;
```
如果输入的是1 2 3，那么输出的是3，因为前两个已经忽略啦。

* 示例2
给定一个字符串"hello, world"，仅保留"world"。（注意：“，”之后有一空格）
```c
sscanf("hello, world", "%*s%s", buf);
printf("%s\n", buf);
```
结果为：world， %*s表示第一个匹配到的 %s 被过滤掉，即 hello,被过滤了，如果没有空格则结果为NULL。

## popen 函数

* 头文件：
`#include <stdio.h>`

* 原型：
`FILE * popen ( const char * command , const char * type )`
`int pclose ( FILE * stream );`

* 作用：
popen() 函数通过创建一个管道，调用 fork 产生一个子进程，执行一个 shell 以运行命令来开启一个进程。这个进程必须由 pclose() 函数关闭，而不是 fclose() 函数。pclose() 函数关闭标准 I/O 流，等待命令执行结束，然后返回 shell 的终止状态。如果 shell 不能被执行，则 pclose() 返回的终止状态与 shell 已执行 exit 一样。

* 参数：
    * type 参数只能是读或者写中的一种，得到的返回值（标准 I/O 流）也具有和 type 相应的只读或只写类型。如果 type 是 "r" 则文件指针连接到 command 的标准输出；如果 type 是 "w" 则文件指针连接到 command 的标准输入。
    * command 参数是一个指向以 NULL 结束的 shell 命令字符串的指针。这行命令将被传到 bin/sh 并使用 -c 标志，shell 将执行这个命令。

* 返回值
    * 如果调用 fork() 或 pipe() 失败，或者不能分配内存将返回NULL，否则返回标准 I/O 流。
    * 返回值是个标准 I/O 流，必须由 pclose 来终止。前面提到这个流是单向的。所以向这个流写内容相当于写入该命令的标准输入；命令的标准输出和调用 popen 的进程相同。与之相反的，从流中读数据相当于读取命令的标准输出；命令的标准输入和调用 popen 的进程相同。

* 示例
```c
#include<stdio.h>
#define _LINE_LENGTH 300
int main(void) 
{
    FILE *file;
    char line[_LINE_LENGTH];
    file = popen("ls", "r");
    if (NULL != file)
    {
        while (fgets(line, _LINE_LENGTH, file) != NULL)
        {
            printf("line=%s\n", line);
        }
    }
    else
    {
        return 1;
    }
    pclose(file);
    return 0;
}
```

## sscanf 函数

* 头文件
`#include<stdio.h>`

* 原型
`int sscanf (const char *str,const char * format,........);`

* 作用
将参数 str 的字符串根据参数 format 字符串来转换并格式化数据。转换后的结果存于对应的参数内。

* 返回值
成功则返回参数数目，失败则返回-1，错误原因存于 errno 中

* 示例

```c
#include<stdio.h>

int main()
{
    int i;
    unsigned int j;
    char input[ ]= "10 0x1b aaaaaaaa bbbbbbbb";
    char s[5];
    sscanf(input,"%d %x %5[a-z] %*s %f",&i,&j,s,s);
    printf("%d %d %s\n",i,j,s);
}
```

执行结果：
`10 27 aaaaa`

## strrchr 函数

* 作用：
strrchr() 函数（在php中）查找字符在指定字符串中从右面开始的第一次出现的位置，如果成功，返回该字符以及其后面的字符，如果失败，则返回 NULL。与之相对应的是 strchr() 函数，它查找字符串中首次出现指定字符以及其后面的字符。

*示例：
```c
#include <string.h>
#include <stdio.h>
int main(void)
{
    char string[20];
    char *ptr, c = 'r';
    strcpy(string, "There are two rings");
    ptr = strrchr(string, c);
    if (ptr)
        printf("The character %c is at position: %s\n", c, ptr);
    else
        printf("The character was not found\n");
    return 0;
}
```
<font color=red>注意</font>：strrchr 返回的指针应当指向 "rings" 里的 'r'，而不是 "There" 或 "are" 里的 'r'。

输出结果：
`The character r is at position：rings`

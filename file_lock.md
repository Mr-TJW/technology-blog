# Linux 文件锁 fcntl 函数详解
----------------------------------------------------------------

> ```c
> #include <unistd.h>
> #include <fcntl.h> 
> int fcntl(int fd, int cmd); 
> int fcntl(int fd, int cmd, long arg); 
> int fcntl(int fd, int cmd, struct flock *lock);
> ```


简介：fcntl() 功能是针对文件描述符提供控制，根据不同的 cmd 对文件描述符可以执行的操作也非常多，用的最多的是**文件记录锁**，也就是 **F_SETLK** 命令，此命令搭配 flock 结构体，对文件进行加解锁操作，例如执行加锁操作，如果不解锁，本进程或者其他进程再次使用 F_SETLK 命令访问同一文件则会告知目前此文件已经上锁，加锁进程退出（正常、异常）后会自行解锁，使用此特性可以实现避免程序多次运行、锁定文件防止其他进行访问等操作。

## 函数功能  

1. 复制一个现有的描述符(cmd = F_DUPFD). 
2. 获得／设置文件描述符标记(cmd = F_GETFD 或 F_SETFD). 
3. 获得／设置文件状态标记(cmd = F_GETFL 或 F_SETFL). 
4. 获得／设置异步I/O所有权(cmd = F_GETOWN 或 F_SETOWN). 
5. 获得／设置记录锁(cmd = F_GETLK , F_SETLK 或 F_SETLKW).

## 函数参数

* fd 文件描述符
* cmd 操作命令

    |cmd|作用|
    |---|---|
    |F_DUPFD|用来查找大于或等于参数 arg 的最小且仍未使用的文件描述词，并且复制参数 fd 的文件描述词。执行成功则返回新复制的文件描述词|
    |F_GETFD|取得 close-on-exec 旗标。若此旗标的 FD_CLOEXEC 位为 0，代表在调用 exec() 相关函数时文件将不会关闭。|
    |F_SETFD|设置 close-on-exec 旗标。该旗标以参数 arg 的 FD_CLOEXEC 位决定|
    |F_GETFL|取得文件描述词状态旗标，此旗标为 open() 的参数 flags。|
    |F_SETFL|设置文件描述词状态旗标，参数 arg 为新旗标，但只允许 O_APPEND、O_NONBLOCK 和 O_ASYNC 位的改变，其他位的改变将不受影响|
    |F_GETLK|取得文件锁定的状态|
    |**F_SETLK**|**设置文件锁定的状态。此时 flcok 结构的 l_type 值必须是 F_RDLCK、F_WRLCK 或 F_UNLCK。操作成功返回0，如果无法建立锁定，则返回-1，错误代码为 EACCES 或 EAGAIN**|
    |F_SETLKW|F_SETLK 作用相同，但是无法建立锁定时，此调用会一直等到锁定动作成功为止。若在等待锁定的过程中被信号中断时，会立即返回-1，错误代码为 EINTR。|

    * lock 类型为结构体flock，设置记录锁的具体状态。
        ```c
        struct flcok 
        { 
            short int l_type;   //锁定的状态

            /* 以下的三个参数用于分段对文件加锁，若对整个文件加锁，则：l_whence = SEEK_SET, l_start = 0, l_len = 0 */
            short int l_whence; //决定l_start位置
            off_t l_start;      //锁定区域的开头位置 
            off_t l_len;        //锁定区域的大小

            pid_t l_pid;        //锁定动作的进程
        };
        ```

## 函数返回值   
* fcntl() 的返回值与命令有关，如果出错，所有命令都返回 -1，错误码保存在 error 中；

* 如果成功则返回某个其他值。下列三个命令有特定返回值：F_DUPFD、F_GETFD、F_GETFL 以及 F_GETOWN；
    |cmd|返回值|
    |---|---|
    |F_DUPFD|返回新的文件描述符|
    |F_GETFD|返回相应标志|
    |F_GETFL|返回一个正的进程ID或负的进程组ID|

* 除此之外的命令成功返回 0，失败返回 -1。

**小技巧** : 为加锁整个文件，通常的方法是将 l_start 说明为 0,l_whence 说明为 SEEK_SET，l_len 说明为 0。

## 使用实例

### 示例一

* 下面首先给出了使用 fcntl() 函数的文件记录锁函数。在该函数中，首先给 flock 结构体的对应位赋予相应的值。
* 接着使用两次 fcntl() 函数分别用于给相关文件上锁和判断文件是否可以上锁，这里用到的cmd值分别为 F_SETLK 和 F_GETLK。

```c

/*lock_set函数*/
void lock_set(int fd, int type)
{   
    time_t curtime;
    struct flock lock;
    lock.l_whence = SEEK_SET;//赋值lock结构体
    lock.l_start = 0;
    lock.l_len = 0;
    while (1){

        curtime = time(NULL);
        lock.l_type = type;

        /*根据不同的type值给文件上锁或解锁*/
        if ((fcntl(fd, F_SETLK, &lock)) == 0){
            if (lock.l_type == F_RDLCK)
                printf("\n%sread lock set by %d\n",asctime(localtime(&curtime)), getpid());
            else if (lock.l_type == F_WRLCK)
                printf("\n%swrite lock set by %d\n",asctime(localtime(&curtime)), getpid());
            else if (lock.l_type == F_UNLCK)
                printf("\n%srelease lock by %d\n",asctime(localtime(&curtime)), getpid());
            return;
        }

        /*判断文件是否可以上锁*/
        fcntl(fd, F_GETLK, &lock);

        /*判断文件不能上锁的原因*/
        if (lock.l_type != F_UNLCK){

            /*/该文件已有写入锁*/
            if (lock.l_type == F_RDLCK)
                printf("\n%sread lock already set by %d\n\n",asctime(localtime(&curtime)),lock.l_pid);

            /*该文件已有读取锁*/
            else if (lock.l_type == F_WRLCK)
                printf("\n%swrite lock already set by %d\n\n",asctime(localtime(&curtime)),lock.l_pid);
            getchar();
        }
    }
}

```

* 这里首先创建了一个hello文件，之后对其上写入锁，最后释放写入锁。

```c

#include <unistd.h>
#include <sys/file.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>
#include

int main(void)
{
    int fd;

    /*首先打开文件*/
    fd = open("hello", O_RDWR | O_CREAT, 0666);
    if (fd < 0){
        perror("open");
        exit(1);
    }
    
    /*给文件上写入锁*/
    lock_set(fd, F_WRLCK);
    getchar();

    /*给文件解锁*/
    lock_set(fd, F_UNLCK);

    getchar();
    close(fd);
    exit(0);
}
```
* 笔者采用文件上锁后由用户键入一任意键使程序继续运行。建议读者开启两个终端，并且在两个终端上同时运行该程序，以达到多个进程操作一个文件的效果。具体步骤如下：
    1. 在这里，笔者首先在终端一中运行该程序；
    2. 在终端一程序运行过程中，打开终端二，也运行该程序；
    3. 此时请读者注意终端二中的第一句，提示已经为锁状态；
    4. 此时在终端一输入任意字符或 Ctrl+C 使程序运行结束，即执行解锁操作；
    5. 终端二中,输入任意字符，将提示已经加锁成功。

终端一：
```c
root@atmel:/mnt/hgfs/Mr Tang/test_use# ./a.out 
Fri Aug  7 14:49:41 2020
write lock set by 3814

Fri Aug  7 14:49:50 2020
release lock by 3814
```

终端二：
```c
root@atmel:/mnt/hgfs/Mr Tang/test_use# ./a.out 
Fri Aug  7 14:49:45 2020
write lock already set by 3814

Fri Aug  7 14:49:54 2020
write lock set by 3815
```

由此可见，写入锁为互斥锁，一个时刻只能有一个写入锁存在。
接下来的程序是测试文件的读取锁:

```c
/*fcntl_read.c测试文件读取锁主函数部分*/
#include <unistd.h>
#include <sys/file.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>
int main(void)
{
    int fd;
    fd = open("hello", O_RDWR | O_CREAT, 0666);
    if (fd < 0){
        perror("open");
        exit(1);
    }

    /*给文件上读取锁*/
    lock_set(fd, F_RDLCK);
    getchar();

    /*给文件解锁*/
    lock_set(fd, F_UNLCK);
    getchar();
    close(fd);
    exit(0);
}
```
操作原理同上面的程序一样

终端一：
```c
root@atmel:/mnt/hgfs/Mr Tang/test_use# ./a.out 
Fri Aug  7 14:59:58 2020
read lock set by 3822

Fri Aug  7 15:02:12 2020
release lock by 3822
```

终端二：
```c
root@atmel:/mnt/hgfs/Mr Tang/test_use# ./a.out 
Fri Aug  7 15:00:04 2020
read lock set by 3823

Fri Aug  7 15:02:15 2020
release lock by 3823
```
读者可以将此结果与写入锁的运行结果相比较，可以看出，读取锁为共享锁，当进程 3822 已设定读取锁后，进程 3823 还可以设置读取锁。

### 示例二

很多时候，我们希望在同一时刻，同一程序仅能启动一个，即防止单个进程同时被被多次启动，可以使用文件锁的方式：

```c
#include <unistd.h>
#include <sys/file.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>

/*******************************************************************
* funcname:	lock_file
* para:	    fd 文件索引
* function:	设置文件锁
* return:	
********************************************************************/
int lock_file(int fd)
{
	struct flock f1;
	f1.l_type = F_WRLCK;
	f1.l_start = 0;
	f1.l_whence = SEEK_SET;
	f1.l_len = 0;
	/* F_SETLK: 设置记录锁 */
	return (fcntl(fd, F_SETLK, &f1));
}

/*******************************************************************
* funcname:	check_process_running
* para:	
* function:	检测当前程序是否被多次启动
* return:	true  此程序目前被多次启动
*           false 此程序目前没有被多次启动
********************************************************************/
bool check_process_running(void)
{
	int fd;
	char buf[16];
 
	/* ORDWR:  以可读写方式打开文件
	 * OCREAT: 如文件不存在,则创建该文件 */
	fd = open("hello", O_RDWR|O_CREAT, S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH);
	if(fd < 0){
		exit(1);
	}
 
	if(lock_file(fd)){
		/* 错误码 errno:
		 * EAGAIN: Try again
		 * EACCES: Permission denied */
		if(errno == EACCES || errno == EAGAIN){
			close(fd);
			return true;
		}
		exit(1);
	}
 
	/* 改变文件大小,清空文件 */
	ftruncate(fd, 0);
	/* 写入进程号,可供管理员等查看 */
	sprintf(buf, "%ld", (long)getpid());
	write(fd, buf, strlen(buf)+1);
	return false;
}

int main(void)
{
    if(check_process_running == true)
    {
        printff("process already running,quit!!!\n");
		exit(1);
    }

    printf("process first running!\r\n");
    getchar();
    return 0;
}

```

## fcntl() 功能详解

1. 复制一个现有的描述符(cmd=F_DUPFD). 
2. 获得／设置文件描述符标记(cmd=F_GETFD或F_SETFD). 
3. 获得／设置文件状态标记(cmd=F_GETFL或F_SETFL). 
4. 获得／设置异步I/O所有权(cmd=F_GETOWN或F_SETOWN). 
5. 获得／设置记录锁(cmd=F_GETLK , F_SETLK或F_SETLKW).

### cmd = F_DUPFD

F_DUPFD  返回值为一个如下描述的(文件)描述符：
* 最小的大于或等于arg的一个可用的描述符
* 与原始操作符一样的某对象的引用
* 如果对象是文件(file)的话，则返回一个新的描述符，这个描述符与arg共享相同的偏移量(offset)
* 相同的访问模式(读，写或读/写)
* 相同的文件状态标志(如：两个文件描述符共享相同的状态标志)
* 与新的文件描述符结合在一起的close-on-exec标志被设置成交叉式访问execve(2)的系统调用

实际上调用 
    `dup(oldfd);` 
等效于:
     `fcntl(oldfd, F_DUPFD, 0);`
而调用
    `dup2(oldfd, newfd);`
等效于
    ```
    close(oldfd)；
    fcntl(oldfd, F_DUPFD, newfd)；
    ```

### cmd = F_GETFD 或 F_SETFD
* F_GETFD 取得与文件描述符 fd 联合的 close-on-exec 标志，类似 FD_CLOEXEC。
* 如果返回值和 FD_CLOEXEC 进行与运算结果是 0 的话，文件保持交叉式访问 exec()，否则如果通过exec运行的话，文件将被关闭(arg 被忽略)。       
* F_SETFD 设置close-on-exec标志，该标志以参数 arg 的 FD_CLOEXEC 位决定，应当了解很多现存的涉及文件描述符标志的程序并不使用常数 FD_CLOEXEC，而是将此标志设置为0(系统默认，在exec时不关闭)或1(在exec时关闭)。   
* 在修改文件描述符标志或文件状态标志时必须谨慎，先要取得现在的标志值，然后按照希望修改它，最后设置新标志值。不能只是执行F_SETFD或F_SETFL命令，这样会关闭以前设置的标志位。 

### cmd = F_GETFL 或 F_SETFL
* F_GETFL 取得 fd 的文件状态标志，如同下面的描述一样(arg 被忽略)，在说明 open 函数时，已说明了文件状态标志。不幸的是，三个存取方式标志 (O_RDONLY , O_WRONLY , 以及 O_RDWR) 并不各占1位，这三种标志的值各是 0、1、2，由于历史原因，这三种值互斥，即一个文件只能有这三种值之一，因此首先必须用屏蔽字O_ACCMODE相与取得存取方式位，然后将结果与这三种值相比较。  
     
* F_SETFL 设置给 arg 描述符状态标志，可以更改的几个标志是：O_APPEND、O_NONBLOCK、O_SYNC、O_ASYNC。而fcntl的文件状态标志总共有7个：O_RDONLY , O_WRONLY , O_RDWR , O_APPEND , O_NONBLOCK , O_SYNC和O_ASYNC。
    可更改的几个标志如下面的描述：
    |标志|含义|
    |---|---|
    |O_NONBLOCK|非阻塞I/O，如果 read(2) 调用没有可读取的数据，或者如果 write(2) 操作将阻塞，则 read 或 write 调用将返回 -1 和 EAGAIN 错误|
    |O_APPEND|强制每次写 (write) 操作都添加在文件大的末尾，相当于 open(2) 的 O_APPEND 标志|
    |O_DIRECT|最小化或去掉 reading 和 writing 的缓存影响。系统将企图避免缓存你的读或写的数据。如果不能够避免缓存，那么它将最小化已经被缓存了的数据造成的影响。如果这个标志用的不够好，将大大的降低性能|
    |O_ASYNC|当I/O可用的时候，允许SIGIO信号发送到进程组，例如：当有数据可以读的时候|

### cmd = F_GETOWN 或 F_SETOWN
* F_GETOWN 取得当前正在接收 SIGIO 或者 SIGURG 信号的进程id 或进程组id，进程组id返回的是负值(arg被忽略)     
* F_SETOWN 设置将接收 SIGIO 和 SIGURG 信号的进程id或进程组id，进程组id 通过提供负值的 arg 来说明(arg 绝对值的一个进程组ID)，否则 arg 将被认为是进程id。

### cmd = F_GETLK 或 F_SETLK 或 F_SETLKW
* 获得／设置记录锁的功能，成功则返回0，若有错误则返回-1，错误原因存于errno。

* F_GETLK 通过第三个参数 arg(一个指向 flock 的结构体)取得第一个阻塞 lock description 指向的锁。取得的信息将覆盖传到fcntl()的flock结构的信息。如果没有发现能够阻止本次锁 (flock) 生成的锁，这个结构将不被改变，除非锁的类型被设置成 F_UNLCK  
  
* F_SETLK 按照指向结构体 flock 的指针的第三个参数 arg 所描述的锁的信息设置或者清除一个文件的 segment 锁。F_SETLK 被用来实现共享(读) 锁 (F_RDLCK) 或独占(写) 锁 (F_WRLCK)，同样可以去掉这两种锁 (F_UNLCK)。如果共享锁或独占锁不能被设置，fcntl() 将立即返回 EAGAIN。

* F_SETLKW 除了共享锁或独占锁被其他的锁阻塞这种情况外，这个命令和 F_SETLK 是一样的。如果共享锁或独占锁被其他的锁阻塞，进程将等待直到这个请求能够完成。当 fcntl() 正在等待文件的某个区域的时候捕捉到一个信号，如果这个信号没有被指定 SA_RESTART, fcntl 将被中断

* 当一个共享锁被 set 到一个文件的某段的时候，其他的进程可以 set 共享锁到这个段或这个段的一部分。共享锁阻止任何其他进程 set 独占锁到这段保护区域的任何部分。如果文件描述符没有以读的访问方式打开的话，共享锁的设置请求会失败。

* 独占锁阻止任何其他的进程在这段保护区域任何位置设置共享锁或独占锁。如果文件描述符不是以写的访问方式打开的话，独占锁的请求会失败。

* l_type 有三种状态：

|状态|含义|
|---|---|
|F_RDLCK|建立一个供读取用的锁定 | 
|F_WRLCK|建立一个供写入用的锁定|
|F_UNLCK|删除之前建立的锁定|

* l_whence 也有三种方式： 

|方式|含义|
|---|---|
|SEEK_SET|以文件开头为锁定的起始位置 | 
|SEEK_CUR|以目前文件读写位置为锁定的起始位置|
|SEEK_END|以文件结尾为锁定的起始位置|
    
* fcntl 文件锁有两种类型：建议性锁和强制性锁
    * 建议性锁是这样规定的：每个使用上锁文件的进程都要检查是否有锁存在，当然还得尊重已有的锁。内核和系统总体上都坚持不使用建议性锁，它们依靠程序员遵守这个规定。
    * 强制性锁是由内核执行的：当文件被上锁来进行写入操作时，在锁定该文件的进程释放该锁之前，内核会阻止任何对该文件的读或写访问，每次读或写访问都得检查锁是否存在。

    系统默认 fcntl 都是建议性锁，强制性锁是非 POSIX 标准的。如果要使用强制性锁，要使整个系统可以使用强制性锁，那么得需要重新挂载文件系统，mount 使用参数 -0 mand 打开强制性锁，或者关闭已加锁文件的组执行权限并且打开该文件的 set-GID 权限位。
    建议性锁只在 cooperating processes 之间才有用。对 cooperating process 的理解是最重要的，它指的是会影响其它进程的进程或被别的进程所影响的进程，举两个例子：
    * 我们可以同时在两个窗口中运行同一个命令，对同一个文件进行操作，那么这两个进程就是 cooperating  processes
    * cat file | sort，那么 cat 和 sort 产生的进程就是使用了 pipe 的 cooperating processes

* 使用 fcntl 文件锁进行 I/O 操作必须小心：进程在开始任何 I/O 操作前如何去处理锁，在对文件解锁前如何完成所有的操作，是必须考虑的。如果在设置锁之前打开文件，或者读取该锁之后关闭文件，另一个进程就可能在上锁/解锁操作和打开/关闭操作之间的几分之一秒内访问该文件。当一个进程对文件加锁后，无论它是否释放所加的锁，只要文件关闭，内核都会自动释放加在文件上的建议性锁(这也是建议性锁和强制性锁的最大区别)，所以不要想设置建议性锁来达到永久不让别的进程访问文件的目的(强制性锁才可以)；强制性锁则对所有进程起作用。

* fcntl 使用三个参数 F_SETLK/F_SETLKW， F_UNLCK和F_GETLK 来分别要求、释放、测试 record locks。record locks 是对文件一部分而不是整个文件的锁，这种细致的控制使得进程更好地协作以共享文件资源。fcntl 能够用于读取锁和写入锁，read lock 也叫 shared lock(共享锁)， 因为多个 cooperating process 能够在文件的同一部分建立读取锁；write lock 被称为 exclusive lock(排斥锁)，因为任何时刻只能有一个 cooperating process 在文件的某部分上建立写入锁。如果 cooperating processes 对文件进行操作，那么它们可以同时对文件加 read lock，在一个 cooperating process 加 write lock 之前，必须释放别的 cooperating process 加在该文件的 read lock 和 wrtie lock，也就是说，对于文件只能有一个 write lock存在，read lock 和 wrtie lock 不能共存。

**下面的例子使用F_GETFL获取fd的文件状态标志。**
```c
#include<fcntl.h>
#include<unistd.h>
#include<iostream>
#include<errno.h>
using namespace std;

int main(int argc,char* argv[])
{
  int fd, var;

  if (argc!=2)
  {
      perror("--");
      cout<<"请输入参数，即文件名！"<<endl;
  }

  if((var=fcntl(atoi(argv[1]), F_GETFL, 0))<0)
  {
     strerror(errno);
     cout<<"fcntl file error."<<endl;
  }

  switch(var & O_ACCMODE)
  {
   case O_RDONLY : cout<<"Read only.."<<endl;
                   break;
   case O_WRONLY : cout<<"Write only.."<<endl;
                   break;
   case O_RDWR   : cout<<"Read wirte.."<<endl;
                   break;
   default  : break;
  }

 if (val & O_APPEND)
    cout<<",append"<<endl;

 if (val & O_NONBLOCK)
    cout<<",noblocking"<<endl;

 cout<<"exit 0"<<endl;

 exit(0);
}
```
       
         
         
          
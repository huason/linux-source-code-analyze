# 进程间通信 - 共享内存
在Linux系统中，每个进程都有独立的虚拟内存空间，也就是说不同的进程访问同一段虚拟内存地址所得到的数据是不一样的，这是因为不同进程相同的虚拟内存地址会映射到不同的物理内存地址上。

但有时候为了让不同进程之间进行通信，需要让不同进程共享相同的物理内存，Linux通过 `共享内存` 来实现这个功能。下面先来介绍一下Linux系统的共享内存的使用。

## 共享内存使用
### 1. 获取共享内存
要使用共享内存，首先需要使用 `shmget()` 函数获取共享内存，`shmget()` 函数的原型如下：
```cpp
int shmget(key_t key, size_t size, int shmflg);
```
* 参数 `key` 一般由 `ftok()` 函数生成，用于标识系统的唯一IPC资源。
* 参数 `size` 指定创建的共享内存大小。
* 参数 `shmflg` 指定 `shmget()` 函数的动作，比如传入 `IPC_CREAT` 表示要创建新的共享内存。

函数调用成功时返回一个新建或已经存在的的共享内存标识符，取决于shmflg的参数。失败返回-1，并设置错误码。

### 2. 关联共享内存
`shmget()` 函数返回的是一个标识符，而不是可用的内存地址，所以还需要调用 `shmat()` 函数把共享内存关联到某个虚拟内存地址上。`shmat()` 函数的原型如下：
```cpp
void *shmat(int shmid, const void *shmaddr, int shmflg);
```
* 参数 `shmid` 是 `shmget()` 函数返回的标识符。
* 参数 `shmaddr` 是要关联的虚拟内存地址，如果传入0，表示由系统自动选择合适的虚拟内存地址。
* 参数 `shmflg` 若指定了 `SHM_RDONLY` 位，则以只读方式连接此段，否则以读写方式连接此段。

函数调用成功返回一个可用的指针（虚拟内存地址），出错返回-1。

### 3. 取消关联共享内存
当一个进程不需要共享内存的时候，就需要取消共享内存与虚拟内存地址的关联。取消关联共享内存通过 `shmdt()` 函数实现，原型如下：
```cpp
int shmdt(const void *shmaddr);
```
* 参数 `shmaddr` 是要取消关联的虚拟内存地址，也就是 `shmat()` 函数返回的值。

函数调用成功返回0，出错返回-1。

### 共享内存使用例子
下面通过一个例子来介绍一下共享内存的使用方法。在这个例子中，有两个进程，分别为 `进程A` 和 `进程B`，`进程A` 创建一块共享内存，然后写入数据，`进程B` 获取这块共享内存并且读取其内容。
* 进程A
```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define SHM_PATH "/tmp/shm"
#define SHM_SIZE 128

int main(int argc, char *argv[])
{
    int shmid;
    char *addr;
    key_t key = ftok(SHM_PATH, 0x6666);
    
    shmid = shmget(key, SHM_SIZE, IPC_CREAT|IPC_EXCL|0666);
    if (shmid < 0) {
        printf("failed to create share memory\n");
        return -1;
    }
    
    addr = shmat(shmid, NULL, 0);
    if (addr <= 0) {
        printf("failed to map share memory\n");
        return -1;
    }
    
    sprintf(addr, "%s", "Hello World\n");
    
    return 0;
}
```

* 进程B
```cpp
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define SHM_PATH "/tmp/shm"
#define SHM_SIZE 128

int main(int argc, char *argv[])
{
    int shmid;
    char *addr;
    key_t key = ftok(SHM_PATH, 0x6666);
    
    char buf[128];
    
    shmid = shmget(key, SHM_SIZE, IPC_CREAT);
    if (shmid < 0) {
        printf("failed to create share memory\n");
        return -1;
    }
    
    addr = shmat(shmid, NULL, 0);
    if (addr <= 0) {
        printf("failed to map share memory\n");
        return -1;
    }
    
    strcpy(buf, addr, 128);
    printf("%s", buf);
    
    return 0;
}
```
测试时先运行进程A，然后再运行进程B，可以看到进程B会打印出 “Hello World”，说明共享内存已经创建成功并且读取。
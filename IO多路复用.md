IO多路复用

C10K问题

```c
/* According to POSIX.1-2001 */
#include <sys/select.h>

/* According to earlier standards */
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
/**
 * nfds fd_set 
 */
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);

作者：后端技术指南针
链接：https://juejin.cn/post/6844904122018168845
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

### 一、两个1024

1. select中存放文件描述符的数组大小FD_SETSIZE为1024
2. 进程的文件描述符上限默认是1024，*正是因为这个原因，select设计时才把数组大小设计为1024*


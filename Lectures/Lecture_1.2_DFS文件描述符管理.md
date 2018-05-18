## DFS 的文件描述符（FD）管理
- dfs怎么支持多个文件系统的，如何对接posix。
- 文件系统的比较。
- 文件系统的一个总体的认识。

### 1. 文件描述符 FD

在RTT 中有多个不同层面的文件描述符,最基本的是文件 fd。

文件 FD 是由一个结构体数组来存储的，其定义在 dfs.c 文件中，
```c
struct dfs_fd fd_table[DFS_FD_MAX];
```
这里的 DFS_FD_MAX 参数最终由 rtconfig.h 中的宏来定义。意思就是系统中允许存在的文件描述符数量。任何类型的文件打开或者使用，都要消耗这个 fd 描述符资源。所以应该根据情况写的稍大一些。
```c
#define DFS_FD_MAX 64
```

这里讨论 fd_get() 函数和 fd_put() 函数的作用。

这个函数将根据文件返回一个文件描述符结构体：

```c
struct dfs_fd *fd_get(int fd)
{
    struct dfs_fd *d;

    fd = fd - DFS_FD_OFFSET;         //减去标准输入输出的偏移量
    if (fd < 0 || fd >= DFS_FD_MAX)  //如果比0小则不可用，如果比fd最大值还大，也不允许，所以返回NULL
        return NULL;

    dfs_lock();                      //获得互斥锁
    d = &fd_table[fd];               //从 fd_table 中获得  fd 文件描述符的指针

    /* check dfs_fd valid or not */
    if (d->magic != DFS_FD_MAGIC)    //检查文件描述符魔数，如果错误则不是文件描述符，返回NULL
    {
        dfs_unlock();
        return NULL;
    }

    /* increase the reference count */
    d->ref_count ++;                 //将改文件的调用次数+1
    dfs_unlock();                    //释放互斥锁

    return d;                        //返回文件描述符的指针
}

void fd_put(struct dfs_fd *fd)
{
    RT_ASSERT(fd != NULL);            //检测传入的描述符结构体指针是否为空

    dfs_lock();
    fd->ref_count --;                 //减少其调用次数

    /* clear this fd entry */
    if (fd->ref_count == 0)           //如果这个文件的调用次数为0，那么在table中清空这个描述符的信息，这里相当于使释放这个描述符了
    {
        memset(fd, 0, sizeof(struct dfs_fd));  
    }
    dfs_unlock();
}

```

- 这里的 `fd_get` 和 `fd_put`    使用了一种计数的惯用手法，在面向对象的系统中，对象之间的协作关系非常复杂，所谓的协作其实就是调用对象的函数或者向对象发送消息，但不管调用函数还是发送消息，总是要通过某种方式知道目标对象才行。而最常见的做法就是保存目标对象的引用（指针）。

- 对象被别人引用了，但是自己可能并不知道。此时如果对象被销毁，对该对象的引用就变成了野指针，系统随时可能因此而崩溃。

- 此时我们可以采用对象引用计数，对象有一个引用计数器，不管谁要引用这个对象，就要把对象的引用计数器+1，如果不再引用了，就把对象的引用计数器-1.当对象的引用计数器被减为0时，说明没有其他对象引用它，该对象就可以安全地销毁了。这样，对象地生命周期就得到了有效地管理。


### 2. 网络 socket 相关 fd

socket 的存储数据结构也是一个结构体数组，在 sockets.c 中定义。
```c
/** The global array of available sockets */
static struct lwip_sock sockets[NUM_SOCKETS];
```
这里的 NUM_SOCKETS 参数最终由 rtconfig.h 中的宏来定义。
```c
#define RT_MEMP_NUM_NETCONN 32
```
意思是系统中最大允许的 socket 连接的数量，这个数值是不允许比上文提到的文件描述符 fd 大的，因为 socket 套接字描述符也是 fd 描述符的一种，使用 Posix 接口来申请 socket 时，首先要申请 fd 描述符的空间，如果这个值大于 fd 的最大值,那么当申请更多的 socket 的时候，必然会出现失败的情况。

```c
int dfs_net_getsocket(int fd)
{
    int sock;
    struct dfs_fd *_dfs_fd; 

    _dfs_fd = fd_get(fd);  //从fd table 返回 fd 结构体
    if (_dfs_fd == NULL) return -1;

    if (_dfs_fd->type != FT_SOCKET) sock = -1;
    else sock = (int)_dfs_fd->data;

    fd_put(_dfs_fd); /* put this dfs fd */
    return sock;
}

int closesocket(int s)
{
    int sock = dfs_net_getsocket(s);  //这里相当于是从文件描述fd获得socket套接字
    struct dfs_fd *d;

    d = fd_get(s);

    /* socket has been closed, delete it from file system fd */
    fd_put(d);                        //先从 table 里面将这个文件描述符删除  这一步是减少其调用次数
    fd_put(d);                        //这一步是清空 table 里面的结构体，因为前面dfs_net_getsocket 里面给调用次数+1，fd_get又+1，所以这里要put两次

    return lwip_close(sock);          //再关闭这个socket链接
}
```
根据以上代码可以看出关闭SOCKET的时候，先是将文件描述符从table中删除，然后再调用lwip提供的接口关闭 socket 连接。
那么现在问题来了，再建立一个socket连接的时候，获取文件描述符fd又是什么流程呢。
list_fd 函数会遍历 fd_table[]，然后将已经使用的 fd 信息打印出来，包括 fd 号码，类型和引用次数。这里打印出的 fd 号码指的是这个 fd 再 table 里面的索引号。

#### 2.1 获取socket fd 
在 POSIX 的 socket 函数中，调用了 fd_new() 函数。
在 fd_new 函数中
```c
int fd_new(void)
{
    struct dfs_fd *d;
    int idx;

    /* lock filesystem */
    dfs_lock();                                   //锁住文件系统

    /* find an empty fd entry */   //在fd_talbe中找到一个空的文件描述符接入点，条件是某个描述符的接入点的调用次数为0
    for (idx = 0; idx < DFS_FD_MAX && fd_table[idx].ref_count > 0; idx++);  

    /* can't find an empty fd entry */ 
    if (idx == DFS_FD_MAX)                                   //直到最后也没有找到，那么到最后返回一个负数
    {
        idx = -(1 + DFS_FD_OFFSET);
        goto __result;
    }

    d = &(fd_table[idx]);                                   //如果找到了一个空的位置那么将这个位置的指针赋给 d
    d->ref_count = 1;                                       //更新调用次数为1
    d->magic = DFS_FD_MAGIC;                                //写入魔数

__result:
    dfs_unlock();                                           //解锁文件系统
    return idx + DFS_FD_OFFSET;                             //返回一个index + fd 偏移值的文件 fd
}      
```

#### 2.2 申请整个 socket fd 的过程

```c
int socket(int domain, int type, int protocol)
{
    /* create a BSD socket */
    int fd;
    int sock;
    struct dfs_fd *d;
    struct lwip_sock *lwsock;

    /* allocate a fd */
    fd = fd_new();     //这里就获取了一个文件描述符 fd
    if (fd < 0)
    {
        rt_set_errno(-ENOMEM);              //如果获取失败，那么返回 -1

        return -1;
    }
    d = fd_get(fd);                         //根据 fd 得到文件描述符结构体的指针

    /* create socket in lwip and then put it to the dfs_fd */
    sock = lwip_socket(domain, type, protocol);     //通过 lwip 接口获得一个socket描述符
    if (sock >= 0)                                  //如果获得成功,那么将相关信息存放到文件描述符 fd 中
    {
        /* this is a socket fd */
        d->type  = FT_SOCKET;
        d->path  = NULL;

        d->fops  = dfs_net_get_fops();

        d->flags = O_RDWR; /* set flags as read and write */
        d->size  = 0;
        d->pos   = 0;

        /* set socket to the data of dfs_fd */
        d->data = (void *) sock;

        lwsock = lwip_tryget_socket(sock);
        rt_list_init(&(lwsock->wait_head));
        lwsock->conn->callback = event_callback;
    }

    /* release the ref-count of fd */
    fd_put(d);

    return fd;                                             //将文件描述符 fd 返回
}
```
# 文件系统数据结构解析
## 1. DFS 框架的组成内容

DFS 框架内部主要包含三个表以及一个文件系统互斥锁用于解决资源冲突的问题：

- `filesystem_operation_table`：这个表的每一个表项表示一个文件系统的对应的一套操作函数及相关属性。
- `filesystem_table`：这个表记录已经挂载的文件系统，即每一个表项表示挂载的一个文件系统。
- `fd_table`：这个表记录当前已经打开的文件描述符。
- `fslock`：文件系统互斥锁，用于文件系统访问的资源冲突问题。

实现代码如下：  
```c
/* Global variables */
const struct dfs_filesystem_ops *filesystem_operation_table[DFS_FILESYSTEM_TYPES_MAX];
struct dfs_filesystem filesystem_table[DFS_FILESYSTEMS_MAX];

/* device filesystem lock */
static struct rt_mutex fslock;

#ifdef DFS_USING_WORKDIR
char working_directory[DFS_PATH_MAX] = {"/"};
#endif

struct dfs_fd fd_table[DFS_FD_MAX];
```

### 1.1  文件系统操作表
`filesystem_operation_table` 是记录各个文件系统操作接口的表，其各个表项的结构如下定义:

```c
/* File system operations */
struct dfs_filesystem_ops
{
    char *name;          // 文件系统的名称
    uint32_t flags;      /* flags for file system operations */

    /* operations for file */
    const struct dfs_file_ops *fops;

    /* mount and unmount file system */
    int (*mount)    (struct dfs_filesystem *fs, unsigned long rwflag, const void *data);
    int (*unmount)  (struct dfs_filesystem *fs);

    /* make a file system */
    int (*mkfs)     (rt_device_t devid);
    int (*statfs)   (struct dfs_filesystem *fs, struct statfs *buf);

    int (*unlink)   (struct dfs_filesystem *fs, const char *pathname);
    int (*stat)     (struct dfs_filesystem *fs, const char *filename, struct stat *buf);
    int (*rename)   (struct dfs_filesystem *fs, const char *oldpath, const char *newpath);
};

struct dfs_file_ops
{
    int (*open)     (struct dfs_fd *fd);
    int (*close)    (struct dfs_fd *fd);
    int (*ioctl)    (struct dfs_fd *fd, int cmd, void *args);
    int (*read)     (struct dfs_fd *fd, void *buf, size_t count);
    int (*write)    (struct dfs_fd *fd, const void *buf, size_t count);
    int (*flush)    (struct dfs_fd *fd);
    int (*lseek)    (struct dfs_fd *fd, off_t offset);
    int (*getdents) (struct dfs_fd *fd, struct dirent *dirp, uint32_t count);

    int (*poll)     (struct dfs_fd *fd, struct rt_pollreq *req);
};

```

由上面的文件操作定义可知，`filesystem_opration_table` 内每一个表项保存的是一个文件系统的一套操作函数，所有类型的操作系统，其操作函数的形式都是一致的。

### 1.2  文件系统挂载表
`filesystem_table` 记录的是当前挂载的文件系统，其每一个表项表示的就是一个文件系统。
文件系统的定义如下：

```c
/* Mounted file system */
struct dfs_filesystem
{
    rt_device_t dev_id;     /* Attached device */

    char *path;             /* File system mount point */
    const struct dfs_filesystem_ops *ops; /* Operations for file system type */

    void *data;             /* Specific file system data */
};
```

### 1.3  文件描述符表
`fd_table` 记录当前打开的文件集合，每一个表项表示一个打开的文件句柄，其结构如下定义:

```c
/* file descriptor */
#define DFS_FD_MAGIC     0xfdfd
struct dfs_fd
{
    uint16_t magic;              /* file descriptor magic number */
    uint16_t type;               /* Type (regular or socket) */

    char *path;                  /* Name (below mount point) */
    int ref_count;               /* Descriptor reference count */

    const struct dfs_file_ops *fops;

    uint32_t flags;              /* Descriptor flags */
    size_t   size;               /* Size in bytes */
    off_t    pos;                /* Current file position */

    void *data;                  /* Specific file system data */
};

struct dfs_file_ops
{
    int (*open)     (struct dfs_fd *fd);
    int (*close)    (struct dfs_fd *fd);
    int (*ioctl)    (struct dfs_fd *fd, int cmd, void *args);
    int (*read)     (struct dfs_fd *fd, void *buf, size_t count);
    int (*write)    (struct dfs_fd *fd, const void *buf, size_t count);
    int (*flush)    (struct dfs_fd *fd);
    int (*lseek)    (struct dfs_fd *fd, off_t offset);
    int (*getdents) (struct dfs_fd *fd, struct dirent *dirp, uint32_t count);

    int (*poll)     (struct dfs_fd *fd, struct rt_pollreq *req);
};
```

### 1.4  文件系统互斥锁

`fslock`  就是一个静态的互斥锁，实现方式如下：
```c
/* device filesystem lock */
static struct rt_mutex fslock;

/**
 * Mutual exclusion (mutex) structure
 */
struct rt_mutex
{
    struct rt_ipc_object parent;                        /**< inherit from ipc_object */

    rt_uint16_t          value;                         /**< value of mutex */

    rt_uint8_t           original_priority;             /**< priority of last thread hold the mutex */
    rt_uint8_t           hold;                          /**< numbers of thread hold the mutex */

    struct rt_thread    *owner;                         /**< current owner of mutex */
};
typedef struct rt_mutex *rt_mutex_t;
```

### 1.5  小结

- 由上面几个小节的内容可知，DFS 框架主要由三张表组成，而 dfs.c, dfs_fs.c, dfs_file.c 这三个源文件主要是围绕着三张表来进行操作的。
- 主要的操作是文件系统的添加、删除和查找等。
- 对具体文件的操作并不在 DFS 框架内实现，而是由中间层的具体文件系统类型来实现。
- 中间层的文件系统向文件系统操作表中注册该文件系统的操作函数。
- 向文件系统挂载表中添加需要挂载的文件系统信息。
- 有了文件系统挂载表、文件系统操作表、文件描述符表这三张表，系统就可以找到对特定文件的操作函数了。
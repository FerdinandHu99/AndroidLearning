# 1 What's Binder?

**Binder** 是 Android 系统中一种高效的进程间通信 (IPC, Inter-Process Communication) 机制。它是 Android 操作系统核心组件之一，由 Linux 内核中的 Binder 驱动实现，结合了消息传递和远程过程调用 (RPC) 的功能。



Binder的主要作用是：

- **服务管理**：Android 的四大组件（Activity、Service、ContentProvider 和 BroadcastReceiver）之间的交互通常通过 Binder 实现。

- **高效通信**：Binder 采用内存映射（memory mapping）的方式，避免了传统 IPC（如管道或套接字）频繁的上下文切换和数据拷贝，因此性能更高。

- **安全性**：每个 Binder 通道都有一个唯一的身份标识（UID 和 PID），Android 使用这个机制验证通信双方的权限。

- **面向对象**：Binder 支持 AIDL（Android Interface Definition Language），可以像调用本地方法一样调用远程服务，使编程变得简单直观。

# 2 Binder核心组件

## 2.1 Binder驱动

① 打开Binder驱动函数

**struct binder_state *binder_open(const char* driver, size_t mapsize)**

```c
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    // 分配binder_state结构体的内存
    bs = malloc(sizeof(*bs));
    if (!bs) {
        // 分配失败，设置errno为ENOMEM
        errno = ENOMEM;
        return NULL;
    }

    // 打开binder驱动设备
    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
    if (bs->fd < 0) {
        fprintf(stderr,"binder:  %s (%s)\n",
                driver, strerror(errno));
        goto fail_open;
    }

    // 检查内核驱动版本与用户空间版本是否匹配
    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr,
                "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }

    bs->mapsize = mapsize;
    // 将设备文件映射到内存
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        // 映射失败，打印错误信息
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```

binder_open函数通过内存映射文件技术，实现内存和binder驱动文件之间的映射关系，并且返回一个地址指向这块内存。

② Binder调用函数（例如向Service Manager发送数据）

```c
/**
 * binder_call
 *  binder 客户端向服务端发送请求
 *  bs: binder_state结构体，描述了binder的状态
 *  msg: binder_io结构体，描述了要发送的数据
 *  reply: binder_io结构体，描述了服务端的回应
 *  target: 服务端的handle
 *  code: 服务端的方法id
 *
 *  该函数将msg中的数据发送给服务端，然后等待服务端的回应，并将回应
 *  存储在reply中
 *
 *  该函数返回0表示成功，否则表示失败
 */
int binder_call(struct binder_state *bs,
                struct binder_io *msg, struct binder_io *reply,
                uint32_t target, uint32_t code)
{
    int res;
    struct binder_write_read bwr;

    struct {
        uint32_t cmd;
        struct binder_transaction_data txn;
    } __attribute__((packed)) writebuf;
    unsigned readbuf[32];

    if (msg->flags & BIO_F_OVERFLOW) {
        fprintf(stderr,"binder: txn buffer overflow\n");
        goto fail;
    }

    writebuf.cmd = BC_TRANSACTION;
    writebuf.txn.target.handle = target;
    writebuf.txn.code = code;
    writebuf.txn.flags = 0;
    writebuf.txn.data_size = msg->data - msg->data0;
    writebuf.txn.offsets_size = ((char*) msg->offs) - ((char*) msg->offs0);
    writebuf.txn.data.ptr.buffer = (uintptr_t)msg->data0;
    writebuf.txn.data.ptr.offsets = (uintptr_t)msg->offs0;

    bwr.write_size = sizeof(writebuf);
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) &writebuf;

    hexdump(msg->data0, msg->data - msg->data0);
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;        
        bwr.read_buffer = (uintptr_t) readbuf;

        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            fprintf(stderr,"binder: ioctl failed (%s)\n", strerror(errno));
            goto fail;
        } 

        
        res = binder_parse(bs, reply, (uintptr_t) readbuf, bwr.read_consumed, 0);
        if (res == 0) {
            return 0;
        }
        if (res < 0) {
            goto fail;
        }
    }

fail:
    memset(reply, 0, sizeof(*reply));
    reply->flags |= BIO_F_IOERROR;
    return -1;
}
```

ioctl函数是应用层函数，该函数的调用在驱动层反应为binder_ioctl函数。

### 2.2 Service Manager

在 Android 系统中，**Service Manager** 是 Binder IPC 机制中的核心组件，负责管理和协调系统服务。它是 Android 平台中的一个守护进程，主要作用是为客户端（应用程序或其他服务）提供一种机制来注册和查找服务。

### 2.3 Binder 代理和 Binder 对象

### 2.4 Binder 映射表

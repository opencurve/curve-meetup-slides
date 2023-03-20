# 前言

本文首先简要介绍下Fuse文件系统以及Fuse文件系统的开发实践，接着会介绍开发fuse文件系统后关于性能优化方面的一些内容。

# Fuse文件系统简介

在内核下开发文件系统不仅难度较高，而且不易调试，并且一旦出问题还会导致系统崩溃，因此用户空间文件系统就应运而生了。用户空间中的文件系统(FUSE)是操作系统的软件接口，允许非特权用户创建自己的文件系统而无需编辑内核代码。这是通过在用户空间中运行文件系统代码来实现的，而FUSE模块仅提供到实际内核接口的桥梁。

# Fuse文件系统开发实践 - “hello world”

## 基础
fuse文件系统的开发简单来讲的话主要为以下几步：

- 实现fuse_lowlevel_ops结构体对应的接口

如下是fuse_lowlevel_ops结构体，不需要实现该结构体的所有函数，按需实现相关函数即可:
```

 struct fuse_lowlevel_ops {
         void (*init) (void *userdata, struct fuse_conn_info *conn);

         void (*destroy) (void *userdata);

         void (*lookup) (fuse_req_t req, fuse_ino_t parent, const char *name);

         void (*forget) (fuse_req_t req, fuse_ino_t ino, uint64_t nlookup);

         void (*getattr) (fuse_req_t req, fuse_ino_t ino,
                          struct fuse_file_info *fi);

         void (*setattr) (fuse_req_t req, fuse_ino_t ino, struct stat *attr,
                          int to_set, struct fuse_file_info *fi);

         void (*readlink) (fuse_req_t req, fuse_ino_t ino);

         void (*mknod) (fuse_req_t req, fuse_ino_t parent, const char *name,
                        mode_t mode, dev_t rdev);

         void (*mkdir) (fuse_req_t req, fuse_ino_t parent, const char *name,
                        mode_t mode);
         // ....  还有其他很多接口
}
```


- 创建fuse session

把上述重定义的fuse_lowlevel_ops作为参数传递给fuse_session_new函数创建session

- 创建挂载

调用fuse_session_mount创建挂载

- 工作

调用fuse_session_loop开始准备接收相应消息，并根据请求类型分别调用fuse_lowlevel_ops中重定义的函数

## 实例

这里我们用一个简单的`hello world`程序来展示下fuse文件系统的开发

```
#define FUSE_USE_VERSION 34

#include <fuse_lowlevel.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <assert.h>

static const char *hello_str = "Hello World!\n";
static const char *hello_name = "hello";

// fuse_lowlevel_op接口函数实现
static int hello_stat(fuse_ino_t ino, struct stat *stbuf)
{
	stbuf->st_ino = ino;
	switch (ino) {
	case 1:
		stbuf->st_mode = S_IFDIR | 0755;
		stbuf->st_nlink = 2;
		break;

	case 2:
		stbuf->st_mode = S_IFREG | 0444;
		stbuf->st_nlink = 1;
		stbuf->st_size = strlen(hello_str);
		break;

	default:
		return -1;
	}
	return 0;
}

// fuse_lowlevel_op接口函数实现
static void hello_ll_getattr(fuse_req_t req, fuse_ino_t ino,
			     struct fuse_file_info *fi)
{
	struct stat stbuf;

	(void) fi;

	memset(&stbuf, 0, sizeof(stbuf));
	if (hello_stat(ino, &stbuf) == -1)
		fuse_reply_err(req, ENOENT);
	else
		fuse_reply_attr(req, &stbuf, 1.0);
}

// fuse_lowlevel_op接口函数实现
static void hello_ll_lookup(fuse_req_t req, fuse_ino_t parent, const char *name)
{
	struct fuse_entry_param e;

	if (parent != 1 || strcmp(name, hello_name) != 0)
		fuse_reply_err(req, ENOENT);
	else {
		memset(&e, 0, sizeof(e));
		e.ino = 2;
		e.attr_timeout = 1.0;
		e.entry_timeout = 1.0;
		hello_stat(e.ino, &e.attr);

		fuse_reply_entry(req, &e);
	}
}

struct dirbuf {
	char *p;
	size_t size;
};

static void dirbuf_add(fuse_req_t req, struct dirbuf *b, const char *name,
		       fuse_ino_t ino)
{
	struct stat stbuf;
	size_t oldsize = b->size;
	b->size += fuse_add_direntry(req, NULL, 0, name, NULL, 0);
	b->p = (char *) realloc(b->p, b->size);
	memset(&stbuf, 0, sizeof(stbuf));
	stbuf.st_ino = ino;
	fuse_add_direntry(req, b->p + oldsize, b->size - oldsize, name, &stbuf,
			  b->size);
}

#define min(x, y) ((x) < (y) ? (x) : (y))

static int reply_buf_limited(fuse_req_t req, const char *buf, size_t bufsize,
			     off_t off, size_t maxsize)
{
	if (off < bufsize)
		return fuse_reply_buf(req, buf + off,
				      min(bufsize - off, maxsize));
	else
		return fuse_reply_buf(req, NULL, 0);
}

// fuse_lowlevel_op接口函数实现
static void hello_ll_readdir(fuse_req_t req, fuse_ino_t ino, size_t size,
			     off_t off, struct fuse_file_info *fi)
{
	(void) fi;

	if (ino != 1)
		fuse_reply_err(req, ENOTDIR);
	else {
        // 当发生readdir调用调用时，返回hello_name文件名
		struct dirbuf b;
		memset(&b, 0, sizeof(b));
		dirbuf_add(req, &b, ".", 1);
		dirbuf_add(req, &b, "..", 1);
		dirbuf_add(req, &b, hello_name, 2);
		reply_buf_limited(req, b.p, b.size, off, size);
		free(b.p);
	}
}

// fuse_lowlevel_op接口函数实现
static void hello_ll_open(fuse_req_t req, fuse_ino_t ino,
			  struct fuse_file_info *fi)
{
	if (ino != 2)
		fuse_reply_err(req, EISDIR);
	else if ((fi->flags & O_ACCMODE) != O_RDONLY)
		fuse_reply_err(req, EACCES);
	else
		fuse_reply_open(req, fi);
}

// fuse_lowlevel_op接口函数实现
static void hello_ll_read(fuse_req_t req, fuse_ino_t ino, size_t size,
			  off_t off, struct fuse_file_info *fi)
{
	(void) fi;

	assert(ino == 2);
    // 当发生read调用时，返回hello_str中的字符串
	reply_buf_limited(req, hello_str, strlen(hello_str), off, size);
}

// fuse_lowlevel_op接口实现
static const struct fuse_lowlevel_ops hello_ll_oper = {
	.lookup		= hello_ll_lookup,
	.getattr	= hello_ll_getattr,
	.readdir	= hello_ll_readdir,
	.open		= hello_ll_open,
	.read		= hello_ll_read,
};

int main(int argc, char *argv[])
{
	struct fuse_args args = FUSE_ARGS_INIT(argc, argv);
	struct fuse_session *se;
	struct fuse_cmdline_opts opts;
	struct fuse_loop_config config;
	int ret = -1;
    // 参数解析
	if (fuse_parse_cmdline(&args, &opts) != 0)
		return 1;
	if (opts.show_help) {
		printf("usage: %s [options] <mountpoint>\n\n", argv[0]);
		fuse_cmdline_help();
		fuse_lowlevel_help();
		ret = 0;
		goto err_out1;
	} else if (opts.show_version) {
		printf("FUSE library version %s\n", fuse_pkgversion());
		fuse_lowlevel_version();
		ret = 0;
		goto err_out1;
	}

	if(opts.mountpoint == NULL) {
		printf("usage: %s [options] <mountpoint>\n", argv[0]);
		printf("       %s --help\n", argv[0]);
		ret = 1;
		goto err_out1;
	}
    // 创建fuse session
	se = fuse_session_new(&args, &hello_ll_oper,
			      sizeof(hello_ll_oper), NULL);
	if (se == NULL)
	    goto err_out1;

	if (fuse_set_signal_handlers(se) != 0)
	    goto err_out2;

    // 挂载
	if (fuse_session_mount(se, opts.mountpoint) != 0)
	    goto err_out3;

	fuse_daemonize(opts.foreground);

    // 开始工作
	/* Block until ctrl+c or fusermount -u */
	if (opts.singlethread)
		ret = fuse_session_loop(se);
	else {
		config.clone_fd = opts.clone_fd;
		config.max_idle_threads = opts.max_idle_threads;
		ret = fuse_session_loop_mt(se, &config);
	}

	fuse_session_unmount(se);
err_out3:
	fuse_remove_signal_handlers(se);
err_out2:
	fuse_session_destroy(se);
err_out1:
	free(opts.mountpoint);
	fuse_opt_free_args(&args);

	return ret ? 1 : 0;
}

```
- 编译

` gcc -Wall hello.c `pkg-config fuse3 --cflags --libs` -o hello`

- 挂载以及使用

```
// 创建挂载目录
root@hostname:~$ mkdir mountpoint
// 挂载
root@hostname:~$ ./hello mountpoint
// 演示
root@hostname:~$ cd mountpoint
root@hostname:~/mountpoint$
root@hostname:~/mountpoint$ ls
hello
root@hostname:~/mountpoint$ cat hello
Hello World!

// 卸载 - 卸载失败，因为挂载点被占用
root@hostname:~/mountpoint$ sudo  umount ~/mountpoint
umount: /home/root/mountpoint: target is busy
        (In some cases useful info about processes that
         use the device is found by lsof(8) or fuser(1).)
// 退出挂载点目录
root@hostname:~/mountpoint$ cd
// 卸载 - 成功
root@hostname:~$ sudo  umount ~/mountpoint
```

# 性能迷雾

前面简单介绍了下fuse的一些基本原理，接下来让我们聊聊性能优化相关的一些问题。在fio测试过程中，发现带宽一直上不去，通过分析client相关日志(或通过systenmtap等工具)可以发现不管fio工具设置的bs size多大，到达fuse应用文件系统的io最多也只有128KB，所以推测在内核层做了io拆分。同时由于内核层有文件锁带来的io串行机制，所以带宽能力会收到限制。关于<font color=red>文件锁，这里敲下小黑板，画个小重点</font>吧：22年6月份，社区加入了一个新feature去支持单文件的并发写，详情见[单文件支持并发写](https://lore.kernel.org/lkml/20220605072201.9237-1-dharamhans87@gmail.com/t/)

那其实这个问题在业内是已经由来已久的问题了，可以追溯到上古时代的[上古issue](https://lkml.org/lkml/2012/7/5/136) , 摘录一段如下:

```
This patch series make maximum read/write request size tunable in FUSE.
Currently, it is limited to FUSE_MAX_PAGES_PER_REQ which is equal
to 32 pages. It is required to change it in order to improve the
throughput since optimized value depends on various factors such
as type and version of local filesystems used and HW specs, etc.

In addition, recently FUSE is widely used as a gateway to connect
cloud storage services and distributed filesystems. Larger data
might be stored in them over networking via FUSE and the overhead
might affect the throughput.

It seems there were many requests to increase FUSE_MAX_PAGES_PER_REQ
to improve the throughput,

```

# 问题解决

从上面的相关issue我们可以看到，大家有讨论过修改fuse请求大小这一问题，但是迟迟没有行动。但是我们迫于业务方的性能方面的压力，所以进行了优化。具体代码见github [fuse带宽优化](https://github.com/wuhongsong/linux/commit/bee0cdf5924f20a96717929cd147e2e5a0c93735)

优化后，我们把fuse_max_pages_per_req设置为了可动态修改的配置项，通过以下命令可以动态调整该值
```
root@pubt1-ceph73:/mnt/test1# echo 1024  > /sys/module/fuse/parameters/fuse_max_pages_per_req
```
该值默认是32，可以通过把该值调大进而提升带宽。通过把该值修改为1024，在我们的poc环境下性能提升数据如下:

|  | 优化前 | 优化后 |
| --- | --- | --- |
| 1M写(带宽) | 70MB/s	 | 192MB/s |
| 1M写(时延) | 233ms	 | 82ms |



# 后续-然鹅

我们的修改是基于4.19内核的，然鹅，接下来的4.20版本就把该配置项的类似修改merge到了内核，详情见 [pr: fuse: add max_pages to init_out](https://github.com/torvalds/linux/commit/5da784cce4308)

```
root@hostname:~/github_linux/linux/fs/fuse$ git show 5da784cce4308
commit 5da784cce4308ae10a79e3c8c41b13fb9568e4e0
Author: Constantine Shulyupin <const@MakeLinux.com>
Date:   Thu Sep 6 15:37:06 2018 +0300

    fuse: add max_pages to init_out

    Replace FUSE_MAX_PAGES_PER_REQ with the configurable parameter max_pages to
    improve performance.

    Old RFC with detailed description of the problem and many fixes by Mitsuo
    Hayasaka (mitsuo.hayasaka.hu@hitachi.com):
     - https://lkml.org/lkml/2012/7/5/136

    We've encountered performance degradation and fixed it on a big and complex
    virtual environment.


```


# 参考文献

 [fuse documentation](https://www.cs.hmc.edu/~geoff/classes/hmc.cs135.201001/homework/fuse/fuse_doc.html#function-purposes)

 [fuse example](https://github.com/libfuse/libfuse/edit/master/example)

 [pr: fuse: add max_pages to init_out](https://github.com/torvalds/linux/commit/5da784cce4308)



# PostgreSQL hooks机制

​	Hooks机制是PostgreSQL扩展里面较小众的一种开发机制，但功能也更加强大，可以开发出其他扩展完成不了的功能。

​	然而这种方法网上的介绍资料并不多，适逢笔者最近需要使用该机制，这里与大家分享一下学习内容和个人的总结。

## Hooks原理和使用

​	Hook其实是一个全局函数指针，在没有扩展加入时默认值为NULL，当PostgreSQL想使用hook时会检查其是否被赋值，被赋值就可以被调用。

​	Hook的类型种类繁多，种类如下：

| Hook                        | 发行版本 |
| --------------------------- | -------- |
| check_password_hook         | 9.0      |
| ClientAuthentication_hook   | 9.1      |
| ExecutorStart_hook          | 8.4      |
| ExecutorRun_hook            | 8.4      |
| ExecutorFinish_hook         | 8.4      |
| ExecutorEnd_hook            | 8.4      |
| ExecutorCheckPerms_hook     | 9.1      |
| ProcessUtility_hook         | 9.0      |
| explain_get_index_name_hook | 8.3      |
| ExplainOneQuery_hook        | 8.3      |
| fmgr_hook                   | 9.1      |
| get_attavgwidth_hook        | 8.4      |
| get_index_stats_hook        | 8.4      |
| get_relation_info_hook      | 8.3      |
| get_relation_stats_hook     | 8.4      |
| join_search_hook            | 8.3      |
| needs_fmgr_hook             | 9.1      |
| object_access_hook          | 9.1      |
| planner_hook                | 8.3      |
| shmem_startup_hook          | 8.4      |

​	Hook扩展函数被封装在共享库中，PostgreSQL调用共享库中的PG_init()函数来加载扩展，调用PG_fini()函数卸载扩展。

​	载入插件流程简单来说，就是先编译好插件，然后修改数据库文件夹下的postgresql.conf文件中shared_preload_libraries的值改为插件名称（多个插件同时使用的情况还没测试过）。重启postgresql后会自动加载插件。

​	举个简单的例子。笔者环境说明一下，postgresql安装在/usr/local/pgsql/下，数据库文件也在该目录下的data文件夹中。插件是放在源码文件下contrib目录下，名称为my_client_auth，包括my_client_auth.c和Makefile文件，一般还有一些.sql或者.control文件，这里的插件比较简单就（暂时）不写了。my_client_auth.c文件如下：

```c
static ClientAuthentication_hook_type next_client_auth_hook = NULL;

void
_PG_fini(void)
{
    ClientAuthentication_hook = next_client_auth_hook;
}

static void my_client_auth(Port *port, int status)
{
 struct stat buf;
 if (next_client_auth_hook)
 (*next_client_auth_hook) (port, status);
 if (status != STATUS_OK)
 return;
 if(!stat("/tmp/connection.stopped", &buf))
 ereport(FATAL, (errcode(ERRCODE_INTERNAL_ERROR),
 errmsg("Connection not authorized!!")));
} 

/* Module entry point */
void
_PG_init(void)
{
 next_client_auth_hook = ClientAuthentication_hook;
 ClientAuthentication_hook = my_client_auth;
}
```

​	Makefile格式如下：

```makefile
MODULE_big = your_hook
OBJS = your_hook.o
ifdef USE_PGXS
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
else
subdir = contrib/your_hook
top_builddir = ../..
include $(top_builddir)/src/Makefile.global
include $(top_srcdir)/contrib/contrib-global.mk
endif
```

​	之前说了这两个文件是在contrib下面，其实也可以放在任意目录下，但编译的时候需要指定USE_PGXS，并且系统环境里要有PG_CONFIG（好麻烦）。为了方便，笔者选择直接在contrib下放文件。

​	转到my_client_auth目录下，make & make install，有需要的还要加sudo满足权限要求。此时，插件已经编译好并且放入了postgresql安装目录中的lib文件夹下。修改shared_preload_libraries = my_client_auth，重启数据库，就将插件安装好啦。

## Hook介绍

​	Hook种类繁多，并且了解这些hook的功能要先了解被替换的函数的作用，所以笔者挑选Executor*_hook、fmgr_hook、planner_hook和shmem_start_up重点讲一下，其余的暂时简略介绍一下。

#### Executor*_hook

#### fmgr_hook

#### planer_hook

#### shmem_start_up



















## 参考资料

- [PostgreSQL wiki](https://wiki.postgresql.org/images/e/e3/Hooks_in_postgresql.pdf)
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



​	





## 参考资料

- [PostgreSQL wiki](https://wiki.postgresql.org/images/e/e3/Hooks_in_postgresql.pdf)
# shmem_startup_hook

​	shmem_startup_hook是用于在PG共享内存中创建全局变量的函数钩。PG的共享内存会被提前分配好，以后需要的共享内存时，都是在这个预分配好的内存中分配。

​	shmem_startup_hook只在CreateSharedMemoryAndSemapohres中被调用，这个函数既可以被postmaster调用，也可以被其他的worker调用，第一次调用会初始化PG内部的一些全局变量，后面再调用时不会再次创建这些全局变量。

​	shmem_startup_hook在CreateSharedMemoryAndSemapohres最后被调用，会按照扩展要求生成用户自定义的数据结构。

​	既然是用来分配共享内存的函数，下面就介绍一下分配共享内存的过程。

## 共享内存分配

​	PG内共享内存的分配是ShmemInitStruct函数来实现的，如果要分配的结构存在，则返回这个内存的指针，否者创建一个新的结构。

​	ShmemInitStruct函数声明为：

```c++
void *
ShmemInitStruct(const char *name, Size size, bool *foundPtr
```

​	对于PG扩展来说，不需要考虑共享内存锁之类的事情，因为锁已经在ShmemInitStruct函数中被使用。

​	比如，要在共享内存中创建一个Example结构的长度为10的数组：

```c++
void 
CreateExampleShmemStruct(void)
{
	size_t required;	//需要分配空间大小
	bool found;			//判断是否被创建过
	int aryLen = 10;
	
	if(pre_shmem_startup)
		(*pre_shmem_startup)();		//检查shmem_startup_hook是被被多次重载过，若被重载过，先执行先前一个函数
	required = ALIGN(sizeof(Example) * aryLen);
	global_example = ShmemInitStruct("Example Structure", required, found);		//global_example是扩展的全局变量
    
    if(found)
        elog(ERROR, "Example Structure exists");
    ...
}
```

​	经过上面的函数就创建了一个Example类型的结构数组，接下来省略的部分就是对这块内存按照开发要求的处理了。

## 小结

​	shmem_startup_hook是一个功能非常明确的函数，实现起来也非常简单，但是对于涉及利用共享内存交互的扩展来说，就需要注意扩展之间的信号量了。
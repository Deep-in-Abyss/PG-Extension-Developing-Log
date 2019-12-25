# ExecutorRun_hook

​	ExecutorRun函数的实质工作是调用了ExecProcNode函数，来获得计划节点获取一个元组，对元组进行相应的处理后，返回处理的结果。

​	我们以ExecNestLoop为例，可以给我们的自定义路径开发一点启发。

## ExecNestLoop

​	大概处理流程：

1、递归得到左右节点的元组。

2、执行节点处理过程。

3、执行选择操作。

4、执行投影操作。

5、返回结果元组指针。

## ExecScan

​	连接节点流程较简单，不如先看一下底层获取元组的实现方式。

​	ExeScan是所有扫描节点的公共执行函数，它需要两个参数，一个是状态节点，另一个是获取扫描元组的函数指针（accessMtd）。

​	状态节点用于记录当前访问节点的状态，accessMtd函数则是对不同类型的扫描函数使用的访问方法。

​	这儿关系有点混乱，我们就以pg-strom来作为例子吧。

### pgstromExecGpuTaskState

​	pgstromExecGpuTaskState函数，有点乱，这篇以后再写吧。




















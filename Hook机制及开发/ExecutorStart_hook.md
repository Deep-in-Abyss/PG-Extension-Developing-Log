# ExecutorStart_hook

​	(*ExecutorStart_hook) (queryDesc, eflags)函数在ExecutorStart函数中，用于替换标准执行开始函数standard_ExecutorStart，接受两个参数，一个是QueryDesc类型的指针queryDesc，包含了生成的计划，另一个是eflags ，记录了掩码，并输出相关的执行信息或结果数据。

​	由于执行器这部分的内容非常繁杂，具体的细节就不再说明了，只会对扩展会用到的量和方法进行介绍。

## standard_ExecutorStart

​	ExecutorStart函数会判断ExecutorStart_hook是否存在，不存在时，则会调用standard_ExecutorStart函数，standard_ExecutorStart的参数也是一个查询描述符和一个掩码。

​	QueryDesc结构中保持着两个关键的结构，一是查询规划产生的PlannedStmt，用于存放生成的路径，而是Estate，用于记录执行器全局状态，包括查询设计的范围表、用到的内存上下文，在节点间传递的元组的全局元组表等等。

​	standard_ExecutorStart做了两件事，一就是构造了Estate，二是构造对应的PlanState树。

​	
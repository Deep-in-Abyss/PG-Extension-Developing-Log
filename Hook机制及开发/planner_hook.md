# planner_hook

​	planner_hook是在查询规划阶段用于截获调用标准查询规划函数standard_planner的钩子。

​	planner_hook在Planner函数中被检查是否被装载，而Planner是PG查询规划的入口。Planner主要功能是接收查询重写后的查询树，并返回PlannStmt结构体，该结构体包含了执行器所需的全部信息，包括计划树（Plan）、子计划树和其他参数。

​	planner_hook函数常常会先调用standard_planner，获得PG的标准规划结果后，再对该结果进行二次更改。

## standard_planner

​	standard_planner先创建了全局的规划状态变量，并检查是否可以并行执行。之后调用subquery_planner开始正式处理查询树。

​	处理一共分为三个部分：预处理、生成路径、生成计划。

#### 预处理

​	预处理主要做的是简化查询树的深度，包括提升子链接、提升子查询、预处理表达式、预处理约束、预处理HAVING子句。

​	提升子链接、子查询简单说来就是将查询语句中的嵌套查询中用到的表，能提出来到FROM子句中的都提出来。例如：

```sql
SELECT D.dname
FROM dept D
WHERE D.deptno IN
	(SELECT E.deptno FROM emp E WHERE E.sal = 10000);
```

​	将其中的子链接提升上来，就变为：

```sql
SELECT D.dname
FROM dept D, (SELECT E.deptno FROM emp E WHERE E.sal = 10000) as Sub
WHERE D.deptno = Sub.deptno
```

​	subquery_planner是递归预处理的，所以上面的结果进一步提升子查询，得到：

```sql
SELECT D.dname
FROM dept D, emp E
WHERE D.deptno = E.deptno AND E.sal = 10000;
```

​	接下来预处理表达式，将一些结果为常量的表达式用常量替换掉，如将“2+2<>4”用False替换掉。

​	简单来说，预处理就是在逻辑上优化了表的访问过程，期间不会涉及任何其他底层算法和物理限制。

#### 生成路径

​	生成路径是生成一个将用到的所有的表，以二叉树的形式连接起来的过程。生成路径的最终结果都放在一个RelOptInfo结构中，生成路径的所有操作几乎都是对这个结构进行的。

```c++
typedef struct RelOptInfo
{
	NodeTag		type;
	RelOptKind	reloptkind;		//关系的类型
	Relids		relids;			//所有用到的表
	double		rows;			//预估行数
	/* per-relation planner control flags */
	bool		consider_startup;
	bool		consider_param_startup;
	bool		consider_parallel;

	struct PathTarget *reltarget;		//目标属性
	/* 物化节点信息 */
	List	   *pathlist;
	List	   *ppilist;
	List	   *partial_pathlist;		/* partial Paths */
	struct Path *cheapest_startup_path;
	struct Path *cheapest_total_path;
	struct Path *cheapest_unique_path;
	List	   *cheapest_parameterized_paths;
	/* 基本表和连接表用到的信息 */
	Relids		direct_lateral_relids;
	Relids		lateral_relids;
	/* 基本表信息 */
	Index		relid;
	Oid			reltablespace;	/* containing tablespace */
	RTEKind		rtekind;		//基本关系的查询类型（普通关系、子查询、函数）
	AttrNumber	min_attr;
	AttrNumber	max_attr;
	Relids	   *attr_needed;
	int32	   *attr_widths;
	List	   *lateral_vars;
	Relids		lateral_referencers;
	List	   *indexlist;
	BlockNumber pages;
	double		tuples;
	double		allvisfrac;
	PlannerInfo *subroot;		/* if subquery */
	List	   *subplan_params; /* if subquery */
	int			rel_parallel_workers;	//并行worker数量
	/* 外表信息 */
	Oid			serverid;
	Oid			userid;
	bool		useridiscurrent;
	struct FdwRoutine *fdwroutine;
	void	   *fdw_private;
	/* used by various scans and joins: */
	List	   *baserestrictinfo;		//基本表的约束信息
	QualCost	baserestrictcost;
	List	   *joininfo;
	bool		has_eclass_joins;
} RelOptInfo;
```

​	其中，pathlist记录了生成该RelOptInfo在某方面较优的路径，类型为Path结构，即路径结构。**路径**描述描述了扫描表的不同方法（T_SeqScan、T_IndexScan等，T_CustomScan也是在这里），和元组排序的不同结果。Path只是一个超类，每个Path节点都对应具体路径节点存放该路径的具体信息，包括连接方式、顺序、以及用到的基本关系的访问方式。

​	路径是树结构，叶子节点表示基本关系的扫描路径，内部节点是连接路径的节点。一个路径实例如下：

```sql
SELECT *
FROM Studnet, Course, SC
WHERE Student.sno = SC.sno AND Course.cno = SC.cno AND sname = 'MM';
```

​	![](.\image\PathExample.png)

​	单个表的访问方式（顺序访问、索引访问、TID访问、自定义方法访问）、两个表间的连接方式（嵌套循环连接、归并连接、Hash连接）以及多个表间的连接顺序（左连接、右连接、布希连接）都有多种，那么**生成的路径都可能是上述访问方式、连接方式、连接顺序的一种组合**。

​	路径生成时，根据启动代价最优路径、总代价最优路径和路径的输出排序键来确定路径的优劣。

​	**NOTICE**：即使两个关系之间没有约束条件，两个表也会被用笛卡尔积连接在一起，但是优先级较低。

​	路径的连接选择是动态规划和遗传算法来实现的。当使用的表数量小于12个时，一般采用的是动态规划。

​	动态规划做的是尝试将k个表以不同的连接方式连接起来，选出其中代价较小和有好排序结果的路径。最终，将所有表连接产生最终的结果。

## planner_hook扩展方式

​	查询规划阶段扩展中，预处理相对独立，一般不会更改。但在生成路径阶段，需要决定表的访问方式和连接方式、连接顺序，这些过程中常常会涉及底层实现，在规划阶段的扩展就是在这些过程中加入自定义的处理函数，也就是新增路径节点种类。

​	之前说过，路径生成有两个步骤，一是确定基本关系的访问方式，二是关系之间的连接方式。

#### 生成自定义的扫描路径

​	为基本表生成访问路径的是set_rel_pathlist函数，该函数会对基本表的类型进行筛选，这些类型包括表、子查询、函数、定值以及其他乱七八糟认不识的。对于普通的表来说，一共会尝试生成三种类型的路径——顺序访问、索引访问、TID访问。create_rel_pathlist函数会对上述类型进行判断并调用相应的函数来处理这些类型。

​	在上述过程完成后，create_rel_pathlist会继续检查**set_rel_pathlist_hook**是否被使用，如果能被调用，则调用该函数生成类型为T_customScan类型的路径，且函数参数和create_rel_pathlist相同，这里就是在扫描阶段的扩展了。

​	以pg-strom来说，create_rel_pathlist_hook被载为gpuscan_add_scan_path函数，接收了全局的规划信息PlannerInfo、基本表的操作信息RelOptInfo、基本表的编号Index和范围表入口RangeTblEntry，最后将生成的路径加入扫描路径中。该函数在检查可以运行后，会为该节点生成一个结构为CustomPath的扫描路径，CustomPath结构如下：

```c++
struct CustomPath
{
	Path		path;
	uint32		flags;	/* mask of CUSTOMPATH_* flags, see
								 * nodes/extensible.h */
	List	   *custom_paths;	/* list of child Path nodes, if any */
	List	   *custom_private;
	const struct CustomPathMethods *methods;
} CustomPath;
```

​	这里面比较重要的量就是path和methods，前者包含了启动时间、总运行时间和是否包含关键字这三个用于判别路径优劣的量，后者则用于实际执行。

#### 生成自定义的连接路径

​	连接方式的生成过程也和扫描过程相似（调用函数的实现的方式有点奇怪啊），由**set_join_pathlist_hook**生成新的自定义连接。

​	set_join_pathlist_hook在add_paths_to_joinrel函数内被调用。add_paths_to_joinrel将分别调用sort_inner_and_outer、match_unsorted_outer、hash_inner_and_outer这三个函数，这三个函数的底层实现才是hash、merge、nested loop方法。

##### 参数意义

​	所有的连接路径函数参数都为root、joinrel、outerrel、innerrel、jointype、extra。

​	extra包含了生成连接路径要用到的量，主要是各种限定条件。

​	jointype指定该连接路径的类型，包括有JOIN_INNER, JOIN_LEFT, JOIN_FULL, JOIN_RIGHT, JOIN_SEMI, JOIN_ANTI, JOIN_UNIQUE_OUTER, JOIN_UNIQUE_INNER。

##### 以sort_inner_and_outer为例

​	sort_inner_and_outer函数创建了一个mergejoin连接路径，具体步骤如下：

1、分别从inner_path和outer_path选出一个**最小代价路径**。 

2、对pathkeys进行排序，保留一些有用的排序给all_pathkeys。

3、对于每种键值排序，尝试产生新的mergejoin_path。

##### pg-strom

​	在pg-strom中，gpujoin_add_join_path担任了这一实现的任务，通过接受全局规划信息PlannerInfo、连接表信息RelOptInfo、外连接表信息RelOptInfo、内连接表信息RelOptInfo、连接类型JoinType和其他信息JoinPathExtraData，并且尝试生成可以在GPU上运行的路径GpuJoinPath。

##### 如何创建一个连接计划

以create_nestloop_plan函数举例。

```c++
static NestLoop *
create_nestloop_plan(PlannerInfo *root,
					 NestPath *best_path)
```

​	先递归得到内外节点计划 ，再排序得到最佳连接约束信息。

​	最后创建嵌套循环计划树（make_nestloop），生成了最终的计划树。其中make_nestloop只填充了NestLoop这个node的内容。

​	其它类型的生成计划也类似，主要过程都包含**提取有效连接信息**、**对连接约束排序**、**分离外连接**、**创建相应类型的计划树**、**复制代价估计**等。

#### 小结

​	postgreSQL在查询规划阶段中还提供了许多这样原本不存在标准实现的函数，但这些函数都只是做一样事情，就是利用hook函数来生成一个新的自定义路径，这种方法生成的路径也可以看作一种和原有的方法平行的方法。

​	至于这些新的路径怎么执行，就是Executor的事情了。

​	到此，这篇关于hook在查询规划阶段的整个流程阶段的作用都有了比较详细的阐述，在实际开发中，一些比较具体的地方还需要研究一下，以后遇到再回来记录。



## 参考资料

-  [彭智勇, PostgreSQL 数据库内核分析](http://www.amazon.cn/PostgreSQL-数据库内核分析-彭智勇/dp/B006BNJNBC) 
-  [PostgreSQL官方文档](https://www.postgresql.org/docs/)










## SS_mem_storage.go 对存储内存中统计信息管理

- stmtKey 包含plan key，txnid。对应size，string方法
- invalidStmtFingerprintID，无效的stmtID
- Container，包含事务和语句的所有统计信息，容器。对应new 方法构建一个container





### IterateAggregatedTransactionStats 、IterateStatementStats、IterateTransactionStats 方法概述

- IterateAggregatedTransactionStats ，提供遍历应用级别所有事务统计信息
  1.   先是dbms_internal.go  1216 行调用
  2. ![image-20230610130740101](C:\Users\c2640\Desktop\sql stats\周六.assets\image-20230610130740101.png)
  3.   后面是ss_local_provider.go  IterateAggregatedTransactionStats 调用 具体细节详见
  4. ![image-20230610130735602](C:\Users\c2640\Desktop\sql stats\周六.assets\image-20230610130735602.png)
  5. 将appname、txnstat信息传入 visitor中
- IterateTransactionStats，提供遍历事务统计信息，具体处理详见MergeApplicationStatementStats 
  1. ​    先是dbms_internal.go，主要就是遍历事务统计信息，输出到虚表中。下同
- IterateStatementStats，提供遍历stmt统计信息，具体处理详见MergeApplicationTransactionStats 







### txnStats结构体方法概述

1. ![image-20230610142334438](C:\Users\c2640\Desktop\sql stats\周六.assets\image-20230610142334438.png)
2. sizeUnsafe 返回txnstats 大小
3. mergeStats 合并txnstats 数据



### stmtStats 结构体方法概述

1. ![image-20230610144417755](C:\Users\c2640\Desktop\sql stats\周六.assets\image-20230610144417755.png)
2. sizeUnsafe 返回stmtstats 大小
3. recordExecStats 记录执行语句统计信息
4. mergeStatsLocked 合并stmt 统计信息数据



### getStatsForStmt 方法概述

- 通过key值拿到stats
- 如果stats为空，构造一个key，创建一个新的entry返回
- 不为空，直接返回



### getStatsForStmtWithKey、getStatsForStmtWithKeyLocked 方法概述

- 通过stmtkey 判断是否存在stats，
- 存在直接返回
- 不存在，创建一个
  -   判断构建的stats条数是否超过内存限制，如果超过了内存限制停止构建
  -  判断内存限制时候，使用了原子操作，https://blog.csdn.net/lml200701158/article/details/119214160



### getStatsForTxnWithKey、getStatsForTxnWithKeyLocked 方法概述

- 同stmt方法操作一致

### NewTempContainerFromExistingStmtStats、ExistingTxnStats方法概述

- 返回当前容器中存在的第一个块中的统计信息，块中所有的entry都是一个appname，



### SaveToLog 方法概述

- 将stats 转换为json格式存在日志中（表）



### Clear 方法概述

- Clear 清除此 Container 中存储的数据，并准备 Container 以供重用。
- 针对以map格式存储的stmt，txn，使用make 方法重新构建



### Free、freeLocked 方法概述

- 使用原子操作，对stmt、txn条目数重置

### NewApplicationStatsWithInheritedOptions 方法概述

- 在sslocal_stats_collector.go，StartExplicitTransaction 中被调用
- 返回一个app stats ，在ss_provider中处理

### MergeApplicationStatementStats 流程概述

#### IterateStatementStats

- IterateStatementStats(context.Context, *IteratorOptions, StatementVisitor) error 参数说明

  ```
  //IterateStatementStats使用StatementVisitor遍历所有收集的语句统计信息。调用者可以通过IteratorOptions参数指定迭代行为，例如排序。StatementVisitor可以返回错误，如果在执行访问者之后返回错误，则
  迭代被中止。
  ```



- 接口具体实现在ss_men_iterater.go文件中，方法名为IterateStatementStats

  1. 根据传入的options变量获取对应的stmt迭代器

  2. 调用iter.Next迭代器方法

  3. Next方法描述

     - 方法意义：接下来更新后续 Cur() 调用返回的当前值。 如果后面的 Cur() 调用有效，则 Next() 返回 true，否则返回 false。
     -  迭代器小标+1，判断是否大于存储的stmtkey大小，如果是，代表超过了存储的大小，返回false
     - 通过下表索引得到stmtkey
     - 通过构造方法，得到stmtstmtFingerprintID
     - 通过getstatsForstmewithkey方法得到存储的stmt统计信息
     - 判断如果stmt统计信息为nil，即不存在数据，那么使用迭代器，迭代下一个stats
     - 拿到从存储中的stmt统计信息，复制给调用的iter对象，如果以上都没有错误，就返回true

     1. 在Next方法中调用 visitor,见步骤4

  4. visitor 即是func(ctx context.Context, statistics *qianpb.CollectedStatementStatistics) error

     -  判断transformer func(*qianpb.CollectedStatementStatistics) 是否为nil，不为nil则将stats传入即是 func(*qianpb.CollectedStatementStatistics) 设置txnid  为 transactionFingerprintID qianpb.TransactionFingerprintID （这部分是在sslocal_stats_collector.go  EndExplicitTransaction的方法中调用）
     -  构建stmtkey，通过存储的map拿到stmtstats，判断是否超过了内存限制，超过那么discardstats，即丢失的stats数据条数+1，返回nil
     - 对stmtstats合并数据
     - 同时，得到逻辑计划的采样样本
     - 判断得到的 得到的采样样本时间是否小于，存储的stats样本时间，是，则将其置为最新的时间即，stats样本时间

  5. 方法执行完成，返回discardstats条数

  6. ![image-20230610112420696](C:\Users\c2640\Desktop\sql stats\周六.assets\image-20230610112420696.png)

  7. ![image-20230610112426983](C:\Users\c2640\Desktop\sql stats\周六.assets\image-20230610112426983.png)





### MergeApplicationTransactionStats 流程概述

#### IterateTransactionStats

- ​	IterateTransactionStats(context.Context, *IteratorOptions, TransactionVisitor) error，	// IterateTransactionStats 使用 TransactionVisitor 遍历所有收集的事务统计信息。 它的行为类似于 IterateStatementStats。

-  接口具体实现在在ss_men_iterater.go文件中，方法名为IterateTransactionStats

  1.  根据传入的options变量获取对应的stmt迭代器
  2. 调用iter.Next迭代器方法
  3. Next方法描述
     -  方法意义：接下来更新后续 Cur() 调用返回的当前值。 如果后面的 Cur() 调用有效，则 Next() 返回 true，否则返回 false。
     - 迭代器小标+1，判断是否大于存储的txnkey大小，如果是，代表超过了存储的大小，返回false
     - 通过下表索引得到txnkey
     - 通过getstatsFortxnwithkey方法得到存储的txn统计信息
     - 判断如果txn统计信息为nil，即不存在数据，那么使用迭代器，迭代下一个stats
     - 拿到从存储中的txn统计信息，复制给调用的iter对象，如果以上都没有错误，就返回true
  4. visitor 即是func(ctx context.Context, statistics *qianpb.CollectedtxnStatistics) error 
     -   构建stmtkey，通过存储的map拿到txnstats，判断是否超过了内存限制，超过那么discardstats，即丢失的stats数据条数+1，返回nil
     - 对txnstats合并数据

  
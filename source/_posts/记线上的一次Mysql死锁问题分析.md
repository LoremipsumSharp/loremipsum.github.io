## 问题描述
最近上线了叫做“F币”的功能，简单说下就是系统会根据用户每天在社区的活跃行为，计算每一个用户的活跃度，最后根据用户每天的总活跃度返还相应的“F币”给用户，用户得到F币之后可以在社区兑换不同的礼品,如下图所示：
![](https://raw.githubusercontent.com/LoremipsumSharp/Images/master/img/BF0B1ABB-F459-4343-BE90-5177FB6583D5.png)

在测试环境和仿真环境运行的好好的，但是上线之后经常接到用户反馈，需要多次点击才能完成领取。查看阿里云日志，提示数据库产生死锁，如下图所示：
![](https://raw.githubusercontent.com/LoremipsumSharp/Images/master/img/9DF85AF7-0840-4323-9B0C-613480533A45.png)

## 问题分析
注意到这里有一排按钮，用户的每次点击，后台都会进行以下两个操作：
1. 更新用户余额（用户表）
2. 生成流水记录 （用户流水记录）

注：用户表和流水表存在主外键关系

以上的两个操作会放在同一个事务完成，并且由于Ef的SaveChanges并不会根据你代码执行的先后次序去更新数据库，当用户以很快的速度从左到右依次点击，存在以下可能：

T1:事务A插入流水，由于存在外键，会对user对应的行上S锁。

T2:事务B插入流水，由于存在外键，会对user对应的行上S锁。

T3:事务A更新User的余额，请求行记录的X锁，被B事务在T2的S锁阻塞

T4:事务B更新User的余额，请求行记录的X锁，被A事务在T1的S锁阻塞

至此死锁产生。

## 问题验证



## 问题解决

用了一个比较简单+暴力的方法：领取接口直接上redis分布式锁。


## 总结与反思

这个项目是使用DDD的思想进行开发的，自然而然地在ORM上面的选型使用了EntityFramework Code First,但是Code Firit在建表的时候会自做主张的在“多”端生成外键。对外键的使用，除开了一部分性能开销，就是上述的死锁问题，后续考虑把这部分重构为DbFirst。

另外值得一提的是，后来对这部逻辑做了压测，发现这部分代码在Sql Server跑是没问题的，因为Sql Server在插入子表的时候不会对父表记录上锁，而Mysql会对父表上锁,所以产生了死锁



## 参考


[DbContext SaveChanges Order of Statement Execution
](https://stackoverflow.com/questions/7335582/dbcontext-savechanges-order-of-statement-execution)

[Deadlock due to Foreign Key constraint](https://bugs.mysql.com/bug.php?id=48652)
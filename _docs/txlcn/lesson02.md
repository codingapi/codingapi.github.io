---
title:  分布式事务从0到1-了解TX-LCN原理
permalink: /docs/txlcn-lesson02/
---

<iframe src="//player.bilibili.com/player.html?aid=80676649&cid=138066500&page=1"
 scrolling="no" border="0" frameborder="no" width="100%" height="600px" framespacing="0" allowfullscreen="true"> </iframe>
本节课讲解的主要内容是TX-LCN分布式事务的原理介绍。

### TX-LCN的核心控制流程

协调控制流程
![](/img/txlcn/yuanli.png)

#### 各种事务模式的原理


|  id  | name  | balacne  | 
|  ----  | ----  | ----  |
| 1  |  A | 200 |
| 2  |  B | 100 |

#### TCC业务处理   
Try Confirm Cancle 
如何实现A转账给B的呢？

```
//尝试方法
function try(){
    //记录日志
    todo save A 转出了 100 元 
    todo save B 转入了 100 元 
    //执行转账
    update amount set balacne = balacne-100 where id = 1
    update amount set balacne = balacne+100 where id = 2
}
//确认方法
function confirm(){
    //清理日志
    clean save A 转出了 100 元 
    clean save B 转出了 100 元 
}

//取消方法
function cancle(){
    //加载日志
    load log A
    load log B

     //退钱
    update amount set balacne = balacne+100 where id = 1
    update amount set balacne = balacne-100 where id = 2    
}


```

特点:  
* 该模式对代码的嵌入性高，要求每个业务需要写三种步骤的操作。   
* 该模式对有无本地事务控制都可以支持使用面广。   
* 数据一致性控制几乎完全由开发者控制，对业务开发难度要求高。   

#### TXC逆向SQL   


![](/img/txlcn/WX20191225-213128@2x.png)
 
特点： 
* 该模式同样对代码的嵌入性低。  
* 该模式仅限于对支持SQL方式的模块支持。  
* 该模式由于每次执行SQL之前需要先查询影响数据，因此相比LCN模式消耗资源与时间要多。   
* 该模式不会占用数据库的连接资源，但中间状态可见   


#### LCN代理连接 

![](/img/txlcn/WX20191225-214414@2x.png)
特点：
* 该模式对代码的嵌入性为低。
* 该模式仅限于本地存在连接对象且可通过连接对象控制事务的模块。
* 该模式下的事务提交与回滚是由本地事务方控制，对于数据一致性上有较高的保障。
* 该模式缺陷在于代理的连接需要随事务发起方一共释放连接，增加了连接占用的时间。


### 负载问题
负载情况下的事务控制，对于无状态的TXC TCC来说是不需要关心事务的。但是对LCN来说需要考虑负载调用同一个模块时若模块不同会可能触发锁的问题。

举例:

目前TX-LCN支持的事务种类有三种，其中LCN模式是会占用资源，详情见LCN模式原理。

若存在这样的请求链,A模块先调用了B模块的one方法，然后在调用了two方法，如下所示：

A ->B.one();
A ->B.two();
假如one与two方法的业务都是在修改同一条数据,假如两个方法的id相同，伪代码如下:
```
void one(id){
   execute => update demo set state = 1 where id = {id} ;
}

void two(id){
   execute => update demo set state = 2 where id = {id} ;
}
```
若B模块做了集群存在B1、B2两个模块。那么就可能出现A分别调用了B1 B2模块，如下:

A ->B1.one();
A ->B2.two();
在这样的情况下业务方将在LCN下会因为资源占用而导致执行失败而回滚事务。为了支持这样的场景，框架提供了重写了rpc的负载模式。

控制在同一次事务下同一个被负载的模块被重复调用时将只会请求到第一次被选中的模块。


### 保障机制与补偿

<iframe src="//player.bilibili.com/player.html?aid=80676836&cid=138067157&page=1" scrolling="no" border="0"
width="100%" height="600px" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe>

#### 超时机制
当业务模块在接受到事务请求，并完成响应以后会自动加入定时任务，等待TM通知，若TM迟迟不通知则触发TC主动请求的状况，若TC没有请求到数据就记录补偿（回滚事务）。
#### TM清理机制
TM全局都在记录着事务的状态消息，只有当TM确认完全都通知到了TC模块才能清楚事务信息，不然将永久保存。

一些特殊的情况介绍：   
1、通知事务的时候通知不到的情况。（需要超时机制，超时机制有分为两种可能 1、联系不上TM不清楚事务状态，2提前询问了TM，业务还没有确认最终状态）   
2、通知事务组执行时没有响应。（1不清楚有没有执行完业务，2不清楚有没有收到消息）   
3、若业务模块死掉了，TM的日志在没有全部确认清楚之前，是不能清理事务数据，TM清理数据需要全部都确认OK方可清理。   

由上述情况可见，需要补偿的情况有   
1、上面的情况1中对联系不上TM的情况需要记录补偿记录。   
2、上面的情况2、3中描述的场景可能会存在业务模块在没有接受到TM消息的时候联系不上了，*若是服务挂了，那么就得需要在下次服务启动的时候通过切面日志来与TM通讯确认状态，然后在执行业务*。若是通讯出现了故障，那么会除非超时机制自动写补偿日志。

由于这样的情况的存在 *若是服务挂了，那么就得需要在下次服务启动的时候通过切面日志来与TM通讯确认状态，然后在执行业务*    
所以只能在切面进入是记录数据，当出现时可通过切面记录来触发业务，然后再补偿事务。  

##### 补偿出现的处理机制
1、自动补偿 (需要开启)   
2、手动补偿 (自行处理删除补偿记录即可)   



<div align="center"><img src="/img/qrcode330.jpg" style="width:300px;" /></div>







 
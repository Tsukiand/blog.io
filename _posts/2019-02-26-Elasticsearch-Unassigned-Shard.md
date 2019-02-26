---
layout: post
title: Elasticsearch集群中出现unassigned shard的原因以及处理方法
date: 2019-02-26
categories: blog
tags: [Elasticsearch,Shard]
description: Elasticsearch异常处理。
---

#Elasticsearch集群中出现unassigned shard的原因以及处理方法 

Elasticsearch集群中出现Unassigned Shard大致有如下几种原因： 

* Shard allocation is purposely delayed
* Too many shards,not enough nodes
* Shard allocation is disabled
* Shard data no longer exists in the cluster
* Low disk watermark
* Multiple elasticsearch versions in cluster
* Hot & Warm node issue
* Cluster level exclude configuration in cluster

我们可以使用Elasticsearch中的API(ES5.X)来获取unassigned shard的原因:(<https://cluster:9200/_cluster/allocation/explain?pretty>)。API将会返回所有的unassigned shard以及出现的原因。

##Shard allocation is purposely delayed(暂时还没有遇到) 

如果data node离开集群，master node会故意延时执行shard reallocation来避免shard rebalancing导致的资源消耗，这样离开的data node能在一定时间内恢复为原来的状态。(默认是一分钟)  
**出现上述情况下的后台日志如下**：  
>[TIMESTAMP][INFO][cluster.routing] [MASTER NODE NAME] delaying allocation for [54] unassigned shards, next check in [1m]. 

**解决方案**:  
```curl -XPUT 'localhost:9200/<INDEX_NAME>/_settings' -d 
'{
        "settings": {
           "index.unassigned.node_left.delayed_timeout": "30s"
        }
}’ -k -u username:passwd```

##Too many shards, not enough nodes 

在node进出集群时候，master node会自动assigned shards，并且要保证shard的多个副本不会存放在相同的node上。也就是说，相同节点上不会出现primary和replica，也不会出现多个replica存放在相同node上。所以shard在节点数目不够的时候会出现unassigned 的状态。为了保证所有shard能顺利存放在node中，需要遵循下面的公式：  
>N >= R + 1 （N表示集群中的数目；R表示集群中index的最大replica数目）

**当集群中的index的replica数目不符合上述公式后就会导致unassigned shard。这时候就可以使用API来修改replica数目解决问题**：  
```curl -X PUT "https://localhost:9200/[index]/_settings" -H 'Content-Type: application/json' -d'
{
    "index.number_of_replicas":0
}
' -u username:passwd -k```

##Shard allocation is disabled 

当我们做rolling-upgrade的时候，升级data node的时候会disable allocation来避免shard re-allocation。但是如果结束的时候你忘记re-enable allocation，那么shard就会出现unassigned shard的问题。  
**解决方案**：  
  
* 点击Cerebro左上角的锁标志就可以实现re-enable shard allocation
* 通过ES API来实现re-enable shard allocation：
```curl -XPUT 'https://localhost:9200/_cluster/settings' -d
'{ "transient":
    { "cluster.routing.allocation.enable" : "all" 
    }
}’ -k -u username:passwd```
		
##Shard data no longer exists in the cluster 

可能出现的shard data no longer exists的几种情况：  

* 某个index存储在node1上并且只有primary shard，不存在副本，这时候node1忽然离开集群。master node可以感知到这个shard（这个shard已经存储在global cluster state file中），但是却无法在集群中定位这个shard data。
* 当集群重启的时候，node会遇到一些问题。正常情况下，node恢复到集群的连接时，node会把存放的shard信息重新sync到master中，那么这些shard就会从unassigned变成 assigned。但是如果这个过程失败了，那么这个node中的shard就会显示unassigned。

**解决方案**：  

* 对出问题的结点进行恢复并且重新加入集群
* 使用ReRoute对shard进行强制allocate(有可能出现数据丢失)
```curl -XPOST 'https://localhost:9200/_cluster/reroute' -d 
'{ "commands" :
      [ { "allocate" : 
          { "index" : "allocate_empty_primary", "shard" : 0, "node": "<NODE_NAME>", "allow_primary": "true","accept_data_loss": true}
      }]
}' -k -u username:passwd```
* 使用Reindex从原数据或者backup中恢复shard数据

##Low Disk Watemark 

mstaer node在集群中没有足够合格（保证足够的磁盘空间，85%使用率是个阈值）node时候，将不会分配shard。这种情况就叫做Low Disk Watermark。Stackoverflow 给出了在你的集群容量超过5T的时候，你就可以提高Low Disk Watermark到90%。  

**解决方案**：  
```curl -XPUT 'https://localhost:9200/_cluster/settings' -d
'{
        "transient": {  
              "cluster.routing.allocation.disk.watermark.low": "90%"
        }
}'  -k -u username:passwd```

如果你想把这个参数在集群重启后还是生效的话，将transient —> persistent. 

##Multiple Elasticsearch Version 

如果有多种Elasticsearch 版本，那么就会出现一下错误，导致unassignd shard。如果是多个版本ES导致的错误时，会出现如下日志：
>[NO(target node version [XXX] is older than source node version [XXX])

##Hot & Warm Node Issue 

目前我们的集群中存在两种data node，warm node 和 hot node。由于最近的数据需要经常查询，hot node存储最近的数据，使用SSD，保证查询效率；而warm node用于存储时间久一点的数据，因为不会经常查询，所以使用机械硬盘。
为了实现上面的hot 和 warm的切换，master node中使用cron job来修改index的template，来保证不同时间的index存储在不同类型的data node上。
之前遇到的事情是，集群中没有warm node，但是index还是会改变，所以旧的index需要存储在warm node中，但是没有warm node，那么就会出现unassigned shard的问题。  
**解决方案**：  

* 修改index的配置(保证index的shard可以落在所有类型的data node上)：
```curl -X PUT "https://localhost:9200/[index]/_settings" -H 'Content-Type: application/json' -d'
{
    "index.routing.allocation.require.box_type":"hot" or “warm"
}
' -u username:passwd -k```  
 
* 修改hot存储的时间和index保存的时间一致，那么就不会存在需要存储在warm node的index。

##Cluster level exclude configuration in cluster 

最近碰到的问题是.security一直无法落在两个node上面，然后尝试各种方法让shard落下还是不行，这时候还是需要使用最开始的那个API获取Unassigned Shard的原因：```https://localhost:9200/_cluster/allocation/explain?pretty``` ，然后就在这个返回值里面发现了问题，返回值显示是cluster 级别的exclude._ip，后面跟着两个IP地址，我一查还真是那个无法allocate shard的两个node。  
首先发现如何查询这个参数的配置：
>curl -XGET  "https://localhost:9200/_cluster/settings" -u username:passwd -k
 
下面是这个API返回的结果，在_ip里面包含了两个ip地址，这样所有的shards就不会落在这两个IP对应的node上。(其实这里面exclude的字段可以有多种方式，_name && _ip && _host 分别是node的名字，node的ip以及node的hostname)

```
{
    "persistent": { }, 
    "transient": {
        "cluster": {
            "routing": {
                "allocation": {
                    "enable": "all", 
                    "exclude": {
                        "_ip": "127.0.0.1,127.0.0.2"
                    }
                }
            }
        }, 
        "indices": {
            "recovery": {
                "max_bytes_per_sec": "10000mb"
            }
        }
    }
}
```

**解决方案**：  
```
Curl -XPUT  "https://localhost:9200/_cluster/settings” -u username:passwd -k -H 'Content-Type: application/json' -d'{"transient":{"cluster.routing.allocation.exclude._ip":""}}'
```  

#结束 

这是我第一次写博客，算是对自己平时学习的一种总结，希望自己能坚持下去！

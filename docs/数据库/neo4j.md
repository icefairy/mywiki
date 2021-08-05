# Neo4j相关

## 关系图谱技术演进

1. 传统关系型数据库扁平化存储+sql查询
   1. 优点：实现简单、无额外学习成本。
   2. 缺点：多级关联数据查询麻烦，且受限于传统关系型数据库的设计原理，对图数据没有专门的优化，在查询多级大量数据时性能不佳。
2. 专用的图数据库例如neo4j
   1. 优点：针对图计算与存储做专门设计提高其性能，cypher专用查询语言较sql能更清楚的表达图之间的关系。
   2. 缺点：需要学习cypher查询语言，数据的增删改查也与传统关系型数据库略有不同。

## 关系图存储查询计算解决方案

> 通过Neo4j实现人员的批量创建，更新（创建后合并），删除，多级关系查询等

## 社区版搭建

```bash
#为了保证性能以及大数据量操作时候不内存溢出，在确保同机器其他组件需求的情况下，尽可能给更多的可用内存
docker run -itd --name neo4j --restart=always -p 7474:7474 -p 7687:7687 -e=NEO4J_dbms_memory_pagecache_size=8G -e=NEO4J_dbms_memory_heap_initial__size=4G -e=NEO4J_dbms_memory_heap_max__size=32G -v /data/neo4j:/data neo4j:4.2.7
#下载apoc插件以提供增强功能支持（比如合并节点），地址：https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases 版本号如果没有完全相同的选择接近的也可以
cd /data/neo4j
wget https://github.com/neo4j-contrib/neo4j-apoc-procedures/releases/download/4.2.0.4/apoc-4.2.0.4-all.jar
#复制apoc*.jar到容器中neo4j目录的plugins文件夹中
docker exec -it neo4j bash
cp data/apoc-*.jar ./plugins/
#修改conf/neo4j.conf配置文件，增加如下两行
dbms.security.procedures.unrestricted=algo.*
dbms.security.procedures.unrestricted=apoc.*
#保存后重启容器
docker restart neo4j
```

## 使用

- 索引建立是非常有必要的，一个索引可以很大程度上降低大规模数据的查询速度。（在对节点按照name属性建立了索引之后，截止我写这篇blog的时候，数据量为节点600w+,关系1100w+，查询一个节点以及相关联的节点关系消耗的时间和数据量很小的时候几乎没有什么差别，基本上都稳定在16~20ms左右。链接：https://www.jianshu.com/p/a2497a33390f ）

> 以下涉及性能的测试相关的机器配置如下:32G内存 8核处理器 机械硬盘

### *批量操作最佳实践*

- 批量创建节点采用(纯新增)：批量创建节点（档案表读取）-推荐
- 批量创建节点采用(新增以及包含更新)：通过证件号合并方式创建-增量更新推荐
- 批量创建关系采用：根据户号创建一家人关系子图事务方式-推荐
- 批量创建关系（复杂类型）采用：通过apoc插件批量方式创建人车关系
- 批量合并实体节点采用：根据证件号码合并节点-cypher方式(依赖apoc插件)
- 批量合并重复关系（主要由合并节点产生）：合并重复关系-cypher方式（依赖apoc插件）

### 批量创建节点

```python
#python dependson:pip install py2neo
from py2neo import Graph
from py2neo.bulk import create_nodes
g = Graph(connectionInfoParam...)
keys = ["name", "age"]
data = [
   ["Alice", 33],
   ["Bob", 44],
   ["Carol", 55],
]
create_nodes(g.auto(), data, labels={"Person"}, keys=keys)
g.nodes.match("Person").count()
```

### 批量创建节点（档案表读取）-纯新增推荐

> 写入1万条耗时7秒，写入操作时，测试机器python和java进程的cpu占用较高，内存合计占用7GB，磁盘读写1-3MB/S，初步推测需要提升写入性能的话需要在cpu方面扩容。 结合车辆档案数据测试结果：21个字段1万条数据写入4.5秒 （人的档案46个字段7秒），将全量字段放到数据仓库中，图数据库中只存必要的查询以及直接显示字段可以有效提升写入速度。

```python
#py2neo 的bulk接口方式
from py2neo import Graph
import psycopg2
from py2neo.bulk import create_nodes,merge_nodes
from itertools import islice
import time
#读取人员档案数据
db=psycopg2.connect("dbname=demo user=postgres password=password hostipaddress port=5432")
cur=db.cursor()
keys="gmsfhm,xm,xbmc,mzmc,xxmc,jggjmc,jgssxmc,csdjgmc,csdssxmc,csdxz,hjdpcsmc,hjdxz,xzz,lxdh,zzmmmc,zw,zy,byzkmc,whcdmc,hyzkmc,zjxymc,hklxmc,hkxzmc,hh,yhzgxdm,yhzgxmc,fqgmsfhm,fqxm,mqgmsfhm,mqxm,pogmsfhm,poxm,idcard_url,province,city,country"
keyss=keys.split(',')
limit=10_000
#固定读取N条数据用来测试合并节点
# sqls="select "+keys+" from czrkjbxx order by hh limit "+str(limit)
#随机读取N条数据
sqls="select "+keys+" from czrkjbxx order by hh asc  offset floor(random()*1000) limit "+str(limit)
nodes=[]
#连接neo4j
graph = Graph("bolt://ipaddress:7687", auth=("neo4j", "neo4jpassword"))
#多少条数据保存一次
batch_size=10000
if batch_size>limit:
   batch_size=limit
try:
   cur.execute(sqls)
   print('query:%s' % sqls)
   data=cur.fetchmany(batch_size)
   ta0=time.time()
   cnt=0
   while data:
       cnt=cnt+1
       print('load data size:%s' % len(data))
       for d in data:
           p={'xm':''}
           for i in range(len(keyss)):
               p[keyss[i]]=d[i]
           nodes.append(p)
           if len(nodes)>=batch_size:
               #启动节点创建
               ta=time.time()
               create_nodes(graph.auto(),nodes,labels={'Person'})
               nodes.clear()
               print("提交 %s 次，耗时:%s秒" % (cnt,round(time.time()-ta)))
           #启动一个自动提交的事务
       if len(nodes)>0:
           create_nodes(graph.auto(),nodes,labels={'Person'})
           nodes.clear()
       data=cur.fetchmany(batch_size)

except(IOError):
   print('发生错误')
print("neo4j中有%s条数据,总耗时:%s秒" % (str(graph.nodes.match('Person').count()),round(time.time()-ta0)))
cur.close()
db.close()
```

### 通过证件号合并方式创建-增量更新推荐

> 1万条耗时283秒

```python
from py2neo import Graph
import psycopg2
from py2neo.bulk import create_nodes,merge_nodes
from itertools import islice
import time
#读取人员档案数据
db=psycopg2.connect("dbname=demo user=postgres password=password host=ip port=5432")
cur=db.cursor()
keys="gmsfhm,xm,xbmc,mzmc,xxmc,jggjmc,jgssxmc,csdjgmc,csdssxmc,csdxz,hjdpcsmc,hjdxz,xzz,lxdh,zzmmmc,zw,zy,byzkmc,whcdmc,hyzkmc,zjxymc,hklxmc,hkxzmc,hh,yhzgxdm,yhzgxmc,fqgmsfhm,fqxm,mqgmsfhm,mqxm,pogmsfhm,poxm,idcard_url,province,city,country"
keyss=keys.split(',')
limit=10_000
#固定读取N条数据用来测试合并节点
# sqls="select "+keys+" from czrkjbxx order by hh limit "+str(limit)
#随机读取N条数据
sqls="select "+keys+" from czrkjbxx order by hh asc  offset floor(random()*1000) limit "+str(limit)
nodes=[]
#连接neo4j
graph = Graph("bolt://ip:7687", auth=("neo4j", "password"))
#多少条数据保存一次
batch_size=10000
if batch_size>limit:
   batch_size=limit
try:
   cur.execute(sqls)
   print('query:%s' % sqls)
   data=cur.fetchmany(batch_size)
   ta0=time.time()
   cnt=0
   while data:
       cnt=cnt+1
       print('load data size:%s' % len(data))
       for d in data:
           p={'xm':''}
           for i in range(len(keyss)):
               p[keyss[i]]=d[i]
           nodes.append(p)
           if len(nodes)>=batch_size:
               #启动节点创建
               ta=time.time()
               merge_nodes(graph.auto(),nodes,('Person','gmsfhm'),labels={'Person'})
               nodes.clear()
               print("提交 %s 次，耗时:%s秒" % (cnt,round(time.time()-ta)))
           #启动一个自动提交的事务
       if len(nodes)>0:
           merge_nodes(graph.auto(),nodes,('Person','gmsfhm'),labels={'Person'})
           nodes.clear()
       data=cur.fetchmany(batch_size)

except(IOError):
   print('发生错误')
print("neo4j中有%s条数据,总耗时:%s秒" % (str(graph.nodes.match('Person').count()),round(time.time()-ta0)))
cur.close()
db.close()
```

### 通过apoc插件批量创建节点

> 写入1000条耗时217秒，不推荐

```python
#通过apoc插件批量创建
from py2neo import Graph
import psycopg2
from itertools import islice
import time
import simplejson

def custJson(d):
   s="{"
   if len(d)>0:
       for k in d:
           if d[k]:
               s=s+k+':"'+d[k]+'",'
           else:
               s=s+k+':"",'
       if len(s)>2:
           return s[0:-1]+'}'
       else:
           return '{}'
   else:
       return '{}'

#读取人员档案数据
db=psycopg2.connect("dbname=demo user=postgres password=password host=ip port=5432")
cur=db.cursor()
keys="gmsfhm,xm,xbmc,mzmc,xxmc,jggjmc,jgssxmc,csdjgmc,csdssxmc,csdxz,hjdpcsmc,hjdxz,xzz,lxdh,zzmmmc,zw,zy,byzkmc,whcdmc,hyzkmc,zjxymc,hklxmc,hkxzmc,hh,yhzgxmc,fqgmsfhm,fqxm,mqgmsfhm,mqxm,pogmsfhm,poxm,idcard_url,province,city,country"
keyss=keys.split(',')
limit=1_000
#固定读取N条数据用来测试合并节点
# sqls="select "+keys+" from czrkjbxx order by hh limit "+str(limit)
#随机读取N条数据
sqls="select "+keys+" from czrkjbxx order by hh asc  offset floor(random()*1000) limit "+str(limit)
nodes="["
#连接neo4j
graph = Graph("bolt://ip:7687", auth=("neo4j", "password"))
#多少条数据保存一次
batch_size=1000
cnt=0
try:
   cur.execute(sqls)
   data=cur.fetchall()
   for d in data:
       p={'xm':''}
       for i in range(len(keyss)):
           p[keyss[i]]=d[i]
       nodes=nodes+custJson(p)+','
       cnt=cnt+1
#         print(nodes)
       if cnt>=batch_size:
           #启动节点创建
           ta=time.time()
           graph.run('call apoc.create.nodes(["Person"],%s)' % (nodes[0:-1]+']'))
           nodes='['
           cnt=0
           print('耗时:%s' % ((time.time()-ta)))
       #启动一个自动提交的事务
   if cnt>0:
       graph.run('call apoc.create.nodes(["Person"],%s)' % (nodes[0:-1]+']'))
       nodes='['
       cnt=0
#     merge_nodes(graph.auto(),)

except(IOError):
   print('发生错误')
print("neo4j中有%s条数据,总耗时:%s" % (str(graph.nodes.match('Person').count()),round(time.time()-ta)))
cur.close()
db.close()
```

### 通过子图方式批量创建节点

> 耗时和py2neo的批量接口create_nodes基本一样

```python
#通过subgraph加事务批量创建节点
from py2neo import Graph,Node,Subgraph
import psycopg2
from itertools import islice
import time
import simplejson

def custJson(d):
   s="{"
   if len(d)>0:
       for k in d:
           if d[k]:
               s=s+k+':"'+d[k]+'",'
           else:
               s=s+k+':"",'
       if len(s)>2:
           return s[0:-1]+'}'
       else:
           return '{}'
   else:
       return '{}'

#读取人员档案数据
db=psycopg2.connect("dbname=demo user=postgres password=password host=ip port=5432")
cur=db.cursor()
keys="gmsfhm,xm,xbmc,mzmc,xxmc,jggjmc,jgssxmc,csdjgmc,csdssxmc,csdxz,hjdpcsmc,hjdxz,xzz,lxdh,zzmmmc,zw,zy,byzkmc,whcdmc,hyzkmc,zjxymc,hklxmc,hkxzmc,hh,yhzgxmc,fqgmsfhm,fqxm,mqgmsfhm,mqxm,pogmsfhm,poxm,idcard_url,province,city,country"
keyss=keys.split(',')
limit=10_000
#固定读取N条数据用来测试合并节点
# sqls="select "+keys+" from czrkjbxx order by hh limit "+str(limit)
#随机读取N条数据
sqls="select "+keys+" from czrkjbxx order by hh asc  offset floor(random()*1000) limit "+str(limit)
nodes="["
#连接neo4j
graph = Graph("bolt://ipaddress:7687", auth=("neo4j", "password"))
tx = graph.begin()
nodes=[]
cur.execute(sqls)
data=cur.fetchall()
ta=time.time()
for line in data:
   oneNode = Node()
   for ki in range(len(keyss)):
       k=keyss[ki]
       oneNode[k]=line[ki]
   nodes.append(oneNode)
nodes=Subgraph(nodes)
tx.create(nodes)
tx.commit()
print('耗时:%s' % ((time.time()-ta)))
```

### 根据证件号码合并节点-cypher方式(依赖apoc插件)

> 2万条数据，实际有1万条不重复数据，执行耗时2秒，推荐 190万数据实际不重复的100万，执行时间4秒

```cypher
--cypher 语法实现
match(p:Person) with p.gmsfhm as sfz,collect(p) as nodelist,count(*) as cnt where cnt>1 call apoc.refactor.mergeNodes(nodelist,{properties:"combine",mergeRels:true}) yield node return node
```

### 合并重复关系-cypher方式（依赖apoc插件）

```sql
MATCH (a:Person)-[r:rel]-(b:Person)
WITH a, b, collect(r) as rels
CALL apoc.refactor.mergeRelationships(rels)
YIELD rel 
RETURN count(rel)
--如果有太多重复数据最好每次只处理一部分，避免全加载到内存中造成内存溢出，如下
match(p:Person) with p.gmsfhm as sfz,collect(p) as nodelist,count(*) as cnt where cnt>1  call apoc.refactor.mergeNodes(nodelist) yield node return node limit 10000
--查询有多少组重复数据
match(p:Person) with p.gmsfhm as sfz,collect(p) as nodelist,count(*) as cnt where cnt>1 return count(nodelist)
```

### 根据证件号合并节点-python方式

> 2万条数据，实际有1万条不重复数据，执行耗时365秒，不推荐

```python
#合并
from py2neo import Graph
import time
from py2neo.bulk import create_nodes,merge_nodes
graph = Graph("bolt://ip:7687", auth=("neo4j", "password"))
print("count:"+str(graph.nodes.match('Person').count()))
nodes=graph.nodes.match('Person')
# mdata=[]
# for n in nodes:
#     mdata.append({'gmsfhm':n['gmsfhm']})

# print(mdata[0])
ta=time.time()
merge_nodes(tx=graph.auto(),data=nodes,merge_key=('Person','gmsfhm'))
print("using time:"+str(round(time.time()-ta)))
print("count:"+str(graph.nodes.match('Person').count()))
```

### 索引（cql）

```sql
CREATE INDEX ON:Person(hh)
--创建多字段的话使用时候必须也传多字段
CREATE INDEX ON:Person(gmsfhm)
--查看索引
:schema
--删除索引
DROP INDEX ON :Person(gmsfhm,hh)
```

### 根据户号创建一家人关系子图事务方式-推荐

> 耗时0.04秒，实测重复执行不会导致建立多个重复关系

```python
#根据户号创建一家人的关系-子图事务方式(推荐)
from py2neo import Graph,Relationship,Subgraph
import time
from py2neo.matching import *
graph = Graph("bolt://ip:7687", auth=("neo4j", "password"))
nodeM=NodeMatcher(graph)
#找一家人
hh='361024002267449'
nodes=nodeM.match('Person',hh=hh).all()
hz=nodeM.match('Person',hh=hh,yhzgxdm='02').first()
print(hz)
if hz:
    print('家人数量:%s' % len(nodes))
    ta=time.time()
    mData=[]
    tx = graph.begin()
    for n in nodes:
        if n['yhzgxdm']!='02':
            mData.append(Relationship(hz,'rel',n,relation=n['yhzgxmc']))
    if len(mData)>0:
        A=Subgraph(relationships=mData)
        graph.create(A)
    tx.commit()
    print('over time:%s' % ((time.time()-ta)))
else:
    print('没有户主的非法数据')
```

### 根据户号创建一家人的关系(不推荐，仅供参考)

> 耗时183秒(不推荐使用，保留下来作为其他可能场景的应用)

```python
#根据户号创建一家人的关系
from py2neo import Graph
import time
from py2neo.bulk import create_nodes,merge_nodes,create_relationships
from py2neo.matching import *
graph = Graph("bolt://ip:7687", auth=("neo4j", "password"))
nodeM=NodeMatcher(graph)
#找一家人
hh='110101007156971'
nodes=nodeM.match('Person',hh=hh).all()
print('家人数量:%s' % len(nodes))
rels="38,68,10,45,51,63,52,99".split(',')
relsmc="儿媳,配偶的曾祖父母,配偶,(外)孙媳妇,父亲,外祖父,母亲,迁出或死亡".split(',')
ta=time.time()
mData=[]
for rel in range(len(rels)):
    relItem=rels[rel]
    relmcItem=relsmc[rel]
    print('relItem:%s mc:%s' % (relItem,relmcItem))
    query="MATCH (p:Person{hh:'%s',yhzgxdm:'02'}),(p2:Person{hh:'%s',yhzgxdm:'%s'}) where p2['gmsfhm'] is not null MERGE (p)-[:rel{relation:'%s'}]->(p2)" % (hh,hh,relItem,relmcItem)
    print("query:%s" % query)
    graph.run(query)
print('over use time:%s r:%s' % (round(time.time()-ta),r))
```

### 通过apoc插件批量方式创建人车关系

> 此方法可以避免同时操作数据过多带来的内存溢出等问题。

```sql
call apoc.periodic.iterate("match(v:Vehicle) return v","match(p:Person{gmsfhm:v.sfzmhm}) merge (p)-[:car {relation:'车主'}]->(v) return v",{batchSize:10000,parallel:true})
```

### 根据指定条件查询N层关系人以及关系数据-cypher方式

```sql
--查询idcard为'idcard'的人为主体，并返回其3层关系人以及关系数据
match data=(na:Person{idcard:'idcard'})-[*1..3]-(nb:Person) return data
```

### 避免内存溢出的清理数据（危险！！！）

```sql
call apoc.periodic.iterate("match(p:Person) return p","delete p return p",{batchSize:10000, parallel:true})
```

### cypher将Label标签作为查询条件

```sql
match data=(na:Person)-[*1..3]-(nb) where any(label in labels(nb) WHERE label in ['Qb', 'Person']) and na.gmsfhm='123456' return data limit 100
```

### cypher将多关系、多标签和可变步长作为查询条件

```sql
--查询idcard为'123456'的人为主体，关系为rel1和rel2 节点标签为label1和label2 且1-3层的关系数据
match data=(na:Person)-[:rel1|rel2*1..3]-(nb) where na.gmsfhm in ['123456']and any(label in labels(nb) WHERE label in ['label1','label2']) return data
```
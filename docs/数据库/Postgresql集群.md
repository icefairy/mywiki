# pg集群搭建
## 创建容器
docker run -d -p 5433:5432 --name pg1 --hostname pg1 --net local livingdocs/postgres:12.4
docker run -d -p 5434:5432 --name pg2 --hostname pg2  --net local livingdocs/postgres:12.4 standby -d "host=pg1 port=5432 user=postgres target_session_attrs=read-write"

# 测试复制
docker exec pg1 psql -c "CREATE TABLE hello (value text); INSERT INTO hello(value) VALUES('world');"
docker exec pg2 psql -c "SELECT * FROM hello;"

# 角色升级（故障恢复）
docker rm pg1 -f
docker exec pg2 psql -c "CREATE TABLE hello1 (value text); INSERT INTO hello1(value) VALUES('world');"
#报错ERROR:  cannot execute CREATE TABLE in a read-only transaction
#升级
docker exec pg2 touch /var/lib/postgresql/data/promote.signal
#执行后等三秒再执行ddl等语句就可以成功，此时以升级为主节点
docker exec pg2 psql -c "CREATE TABLE hello1 (value text); INSERT INTO hello1(value) VALUES('world');"
#增加从节点
docker run -d -p 5433:5432 --name pg1 --hostname pg1  --net local livingdocs/postgres:12.4 standby -d "host=pg2 port=5432 user=postgres target_session_attrs=read-write"
#测试从节点
docker exec pg1 psql -c "SELECT * FROM hello;"
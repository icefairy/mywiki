# 使用基于容器的pyflink

```bash
#下面指定的路径/tmp/flink-checkpoints不用创建，程序会自动创建，如果要手动创建注意赋权
mkdir /data/{flinkjobmgr,flinktm}
docker network create pynet
#jobmanager
FLINK_PROPERTIES="jobmanager.rpc.address: jobmanager
state.backend: filesystem
web.submit.enable: false
state.checkpoints.num-retained: 1
state.checkpoints.dir: file:///tmp/flink-checkpoints
state.savepoints.dir: file:///tmp/flink-savepoints
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: 6553500
restart-strategy.fixed-delay.delay: 10s
table.dynamic-table-options.enabled: true
jobmanager.memory.process.size: 8192m
historyserver.archive.fs.dir: file:///tmp/flink-cmpdir
jobmanager.archive.fs.dir: file:///tmp/flink-cmpdir
"
docker run  -itd   --name=jobmanager -v /data/flinkjobmgr:/tmp --hostname jobmanager  --net pynet -p 8081:8081 -p 8082:8082 -p 6123:6123 --restart=always  -e FLINK_PROPERTIES="${FLINK_PROPERTIES}"     icefairy/flink:1.16.1_cdc jobmanager
#批量构建taskmanager
mkdir /data/flinktm -p;
chmod 777 -R /data/flinktm;
for i in {1..6};do
echo create $i;
FLINK_PROPERTIES="jobmanager.rpc.address: jobmanager
taskmanager.memory.process.size: 30720m
state.backend: filesystem
state.checkpoints.num-retained: 1
state.backend.incremental: false
state.checkpoints.dir: file:///tmp/flink-checkpoints
state.savepoints.dir: file:///tmp/flink-savepoints
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: 6553500
restart-strategy.fixed-delay.delay: 10s
table.dynamic-table-options.enabled: true
taskmanager.numberOfTaskSlots: 6
"
docker run -itd --name tm$i -v /data/flinktm:/tmp --hostname taskmanager$i --net pynet --restart=always -e FLINK_PROPERTIES="${FLINK_PROPERTIES}" icefairy/flink:1.16.1_cdc taskmanager

done
docker ps 

#sql-client
FLINK_PROPERTIES="jobmanager.rpc.address: jobmanager"
#下面命令执行后再执行`./bin/sql-client.sh`即可
docker run -it --rm -v `pwd`:/data --net pynet -e TZ= -e FLINK_PROPERTIES="${FLINK_PROPERTIES}" flink:1336cdc bash

```

## 附加构建

```Dockerfile
from flink:1.13.6-scala_2.11-java8
add sources.list /etc/apt/sources.list
copy extlib/*.jar /opt/flink/lib/
RUN apt-get update -y && \
apt-get install -y build-essential libssl-dev zlib1g-dev libbz2-dev libffi-dev && \
wget https://registry.npmmirror.com/-/binary/python/3.7.9/Python-3.7.9.tgz && \
tar -xvf Python-3.7.9.tgz && \
cd Python-3.7.9 && \
./configure --without-tests --enable-shared && \
make -j6 && \
make install && \
ldconfig /usr/local/lib && \
cd .. && rm -f Python-3.7.9.tgz && rm -rf Python-3.7.9 && \
ln -s /usr/local/bin/python3 /usr/local/bin/python && \
apt-get clean && \
rm -rf /var/lib/apt/lists/*

COPY apache-flink*.tar.gz /
RUN pip3 config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple && pip3 install /apache-flink-libraries-1.13.6.tar.gz && pip3 install /apache-flink-1.13.6.tar.gz && rm /*.tar.gz -f
```

```
flink的py依赖：https://archive.apache.org/dist/flink/flink-1.13.6/python/
python源码：https://registry.npmmirror.com/binary.html?path=python/3.7.9/
cdc依赖：https://ververica.github.io/flink-cdc-connectors/master/
```

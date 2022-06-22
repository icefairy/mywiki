#IO测试#

* 测试随机读取
  `fio -filename=/tmp/test_randread -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=30G -numjobs=10 -runtime=60 -group_reporting -name=mytest`
* 测试随机写入
  `fio -filename=/tmp/test_randread -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=30G -numjobs=10 -runtime=600 -group_reporting -name=mytest`
* 顺序读
  ` fio -filename=/tmp/test_randread -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=30G -numjobs=10 -runtime=60 -group_reporting -name=mytest`
* 顺序写
  `fio -filename=/data/test_randread -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=16k -size=30G -numjobs=10 -runtime=600 -group_reporting -name=mytest`
* 混合随机读写
  `fio -filename=/tmp/test_randread -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=30G -numjobs=10 -runtime=600 -group_reporting -name=mytest -ioscheduler=noop`


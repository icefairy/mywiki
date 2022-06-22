```
#在要测试的机器安装iperf3
yum install iperf3 -y
#在A机器执行监听
ipA:iperf3 -s
#在B机器执行发送
ipB:iperf3 -c ipA -b 10000M -n 100G
```
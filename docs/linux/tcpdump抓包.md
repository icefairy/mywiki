# tcpdump抓包
```bash
#抓取tcp协议的8120端口数据
tcpdump -X -vv -n -nn -i eth0 tcp port 8120 
```
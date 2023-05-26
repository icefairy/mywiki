<!-- 通过hyper-v管理器创建一个外部交换机然后在%USERPROFILE%/.wslconfig填写
[wsl2]
networkingMode=bridged
vmSwitch=WSLBridge
ipv6=true

重启wsl
wsl --shutdown && wsl  -->

netsh interface portproxy add v4tov4 listenport=8889 listenaddress=0.0.0.0 connectport=8888 connectaddress=172.28.110.0
netsh interface portproxy add v4tov4 listenport=7023 listenaddress=0.0.0.0 connectport=22 connectaddress=172.28.110.0
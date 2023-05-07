# 服务节点
docker run -dit --name n2n_supernode -e SUPERNODE_PORT=10000 --network=host --restart=always forgaoqiang/n2n_supernode:latest

# 客户节点
docker run -dit --network=host --privileged --name=n2n_edge  --restart=always \
-e EDGE_PORT=10001 \
-e EDGE_SUPERNODE=49.235.69.218:10000 \
-e EDGE_TUN_NAME=edge0 \
-e EDGE_IP=10.4.0.12 \
-e EDGE_NETMASK=255.255.255.0 \
-e EDGE_COMMUNITY=n2nnet123 \
 -e EDGE_KEY=n2nnet123 \
forgaoqiang/n2n_edge

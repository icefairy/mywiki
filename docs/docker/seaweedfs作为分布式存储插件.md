## 安装插件

### 在线安装

```bash
#HOST的值是filer的地址和端口，不带http前缀
docker plugin install --alias seaweedfs katharostech/seaweedfs-volume-plugin HOST=localhost:8888 
#安装过程中会询问是否允许申请的权限，输入Y回车进行确认
docker plugin ls #可以看到如下结果
ID                  NAME                 DESCRIPTION                         ENABLED
4a08a23cf2eb        seaweedfs:latest     SeaweedFS volume plugin for Docker  true
```

### 离线安装

在有网络的环境下安装好插件，将插件目录（`/var/lib/docker/plugin/*`）直接打包起来拿到离线环境下解压到对应目录，然后`systemctl restart docker`来重启容器后端服务即可通过`docker pulgin ls`看到该插件了。

## 创建存储卷

```bash
#创建一个持久化存储卷
docker volume create --driver seaweedfs weed-vol
#该目录存储于：http://localhost:8888/docker/volumes/weedvol/

```

## 使用

```bash
docker run -it --rm -v weed-vol:/data alpine sh
cd /data
cp -r /etc/* ./
ls -alh
du . -hs

```

## 注意事项

* 生产环境中建议seaweedfs集群使用多个volume来提升并发读写速度。
* 如果想将docker的volume绑定到不同的seaweedfs filer的话以另外的名称再安装一个插件，创建容器存储卷时候使用另一个名字作为`--driver`的值即可。
* 插件通过`host`主机网络接口进行通信。
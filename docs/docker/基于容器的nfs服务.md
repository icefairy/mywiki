```bash
docker run -itd --restart=always -v /data/nfsroot:/data --net host -e SHARED_DIRECTORY=/data -e FILEPERMISSIONS_UID=1 -e FILEPERMISSIONS_GID=1 -e FILEPERMISSIONS_MODE=777  openebs/nfs-server-alpine 

```
新版本中加的特性，启动容器时候直接传`--security-opt seccomp=unconfined `即可绕过，详情可以参考：https://docs.docker.com/engine/security/seccomp/
```bash
docker run --rm -it --security-opt seccomp=unconfined debian:jessie \
    unshare --map-root-user --user sh -c whoami
    ```
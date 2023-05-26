# Centos7更新openssh修复漏洞
``bash
yum -y install wget tar gcc make dropbear
#修改dropbear的端口并启动用于升级期间的远程访问，端口为8022
/usr/lib/systemd/system/dropbear.service
ExecStart=/usr/sbin/dropbear -E -R -p 8022 -F $OPTIONS
systemctl start dropbear
systemctl enable dropbear
#然后通过8022进行ssh登录，确定ok后进行后续步骤,后续操作都通过8022的ssh链接进行操作，直到最后22默认的ssh升级完毕
mkdir /tmp/updatessh
cd /tmp/updatessh
wget https://mirrors.aliyun.com/pub/OpenBSD/OpenSSH/portable/openssh-9.1p1.tar.gz
wget https://www.zlib.net/zlib-1.2.12.tar.gz
wget https://www.openssl.org/source/openssl-1.1.1q.tar.gz
tar zxvf openssh-9.1p1.tar.gz
tar zxvf openssl-1.1.1q.tar.gz
tar zxvf zlib-1.2.12.tar.gz
cd zlib-1.2.12
./configure --prefix=/usr/local/zlib
make && make install
cd ../openssl-1.1.1q
./config --prefix=/usr/local/ssl -d shared
make && make install
echo '/usr/local/ssl/lib' >> /etc/ld.so.conf
ldconfig -v
cd ../openssh-9.1p
./configure --prefix=/usr/local/openssh --with-zlib=/usr/local/zlib --with-ssl-dir=/usr/local/ssl
make && make install
#卸载自带的
yum remove openssh
#启动前修改配置项文件 /usr/local/openssh/etc/sshd_config 中的以下项目，以免无法登录
PermitRootLogin yes
PubkeyAuthentication yes
PasswordAuthentication yes
cp contrib/redhat/sshd.init /etc/init.d/sshd
chkconfig --add sshd
cp /usr/local/openssh/etc/sshd_config /etc/ssh/sshd_config
cp /usr/local/openssh/sbin/sshd /usr/sbin/sshd
cp /usr/local/openssh/bin/ssh /usr/bin/ssh
cp /usr/local/openssh/bin/ssh-keygen /usr/bin/ssh-keygen
cp /usr/local/openssh/etc/ssh_host_ecdsa_key.pub /etc/ssh/ssh_host_ecdsa_key.pub
systemctl start sshd
systemctl status sshd
#可以正常登录 完成
#可选 清理dropbear服务
systemctl stop dropbear
systemctl disable dropbear
```
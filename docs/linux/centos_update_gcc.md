# CentOS完美升级gcc版本方法

在某些应用场景中，需要特定的gcc版本支持，但是轻易不要去编译gcc，我这里推荐使用红帽提供的开发工具包来管理gcc版本，这样做的好处是随时切换版本，并且可以并存多个版本，不破坏原有gcc环境。

红帽官方Developer Toolset文档地址：https://access.redhat.com/documentation/en-us/red_hat_developer_toolset/8/

devtoolset对应gcc的版本:

devtoolset-3对应gcc4.x.x版本 devtoolset-4对应gcc5.x.x版本 devtoolset-6对应gcc6.x.x版本 devtoolset-7对应gcc7.x.x版本 1、安装devtoolset包

yum install centos-release-scl yum install devtoolset-4 2、激活gcc版本，使其生效

scl enable devtoolset-4 bash

source /opt/rh/devtoolset-4/enable 注意：此时通过gcc --version命令可以看到，gcc版本已经变成5.3.1，值得注意的是这仅仅在当前bash生效，如果需要永久生效，可以请自行添加环境变量。
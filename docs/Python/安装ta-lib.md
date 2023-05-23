## windows
这里找到你对应的python版本和系统版本下载然后pip安装本地包即可
https://www.lfd.uci.edu/~gohlke/pythonlibs/#ta-lib

## linux
```bash
wget http://prdownloads.sourceforge.net/ta-lib/ta-lib-0.4.0-src.tar.gz
tar -xzf ta-lib-0.4.0-src.tar.gz
cd ta-lib/
./configure --prefix=/usr
make
sudo make install
#安装最新版本
pip install ta-lib
```
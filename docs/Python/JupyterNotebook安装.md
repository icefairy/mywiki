# Jupyter Notebook 安装
```bash
#安装Anaconda
#按照https://mirror.tuna.tsinghua.edu.cn/help/anaconda/ 设置加速镜像
jupyter notebook --generate-config
conda create -n ice python=3.7
conda install nb_conda
jupyter notebook password

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

pip install jupyter_contrib_nbextensions jupyter_nbextensions_configurator
jupyter contrib nbextension install
jupyter nbextensions_configurator enable
jupyter notebook
``` 
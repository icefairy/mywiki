# 基于alpine的openjdk8无法生成验证码的解决
> 原因在于授权问题openjdk没有自带awt字体相关内容，自己安装一下即可。

```bash
apk add fontconfig 
apk add --update ttf-dejavu
fc-cache --force
#然后重启程序即可
```
# 在linux通过docker运行微信
docker pull bestwu/wechat
vim wechat.sh
docker run -d --name wechat --device /dev/snd --ipc="host" \
 -v /tmp/.X11-unix:/tmp/.X11-unix \
 -v /opt/data/WeChatFiles:/WeChatFiles \
 -e DISPLAY=unix$DISPLAY \
 -e XMODIFIERS=@im=fcitx \
 -e QT_IM_MODULE=fcitx \
 -e GTK_IM_MODULE=fcitx \
 -e AUDIO_GID=`getent group audio | cut -d: -f3` \
 -e GID=`id -g` \
 -e UID=`id -u` \
bestwu/wechat



wget https://dldir1.qq.com/weixin/Windows/WeChatSetup.exe
deepin.com.qq.office


https://com-store-packages.uniontech.com/appstore/pool/appstore/c/com.qq.weixin.deepin/com.qq.weixin.deepin_3.2.1.154deepin14_i386.deb
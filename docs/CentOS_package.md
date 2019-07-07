CentOS Pakages
===================
设置CentOS软件源

# 本地软件源设置和redhat相同
请参考：[[redhat 软件源设置]](redhat_package.md)

# 在线源

直接下载到本地即可
```
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.huaweicloud.com/repository/conf/CentOS-7-anon.repo
```


## 问题解决
如果出现repodata/repomd.xml Error 404
```
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * epel: fedora.cs.nctu.edu.tw
https://mirrors.huaweicloud.com/centos/7/os/aarch64/repodata/repomd.xml: [Errno 14] HTTPS Error 404 - Not Found
Trying other mirror.
To address this issue please refer to the below wiki article

https://wiki.centos.org/yum-errors

If above article doesn't help to resolve this issue please use https://bugs.centos.org/.

https://mirrors.huaweicloud.com/centos/7/extras/aarch64/repodata/repomd.xml: [Errno 14] HTTPS Error 404 - Not Found
Trying other mirror.
https://mirrors.huaweicloud.com/centos/7/updates/aarch64/repodata/repomd.xml: [Errno 14] HTTPS Error 404 - Not Found
```
解决办法修改CentOS-Base.repo, baseurl中的$releasever替换为7，centos替换为centos-altarch
```
baseurl=https://mirrors.huaweicloud.com/centos/$releasever/os/$basearch/
修改为：
baseurl=https://mirrors.huaweicloud.com/centos-altarch/7/updates/$basearch/
```
go
=======
编译安装golang
```
yum install golang
git clone https://github.com/golang/go
cd src
./all.bash
```

如果编译成功
```
##### ../test/bench/go1
testing: warning: no tests to run
PASS
ok      _/home/me/go/test/bench/go1     1.914s

##### ../test

##### API check
Go version is "go1.12.7", ignoring -next /home/me/go/api/next.txt

ALL TESTS PASSED
---
Installed Go for linux/arm64 in /home/me/go
Installed commands in /home/me/go/bin
*** You need to add /home/me/go/bin to your PATH.
```
设置path，注意替换成自己的路径
```
[me@centos src]$ go version
go version go1.11.5 linux/arm64
[me@centos src]$ export PATH=/home/me/go/bin:$PATH
[me@centos src]$ echo $PATH
/home/me/go/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/me/.local/bin:/home/me/bin
[me@centos src]$
[me@centos src]$ go version
go version go1.12.7 linux/arm64
[me@centos src]$
```


# 问题记录

### 从源码安装go不成功 
```
[2019-08-13 17:18:11]  [me@centos src]$ ./all.bash 
[2019-08-13 17:18:16]  Building Go cmd/dist using /home/me/go1.4.
[2019-08-13 17:18:16]  ERROR: Cannot find /home/me/go1.4/bin/go.
[2019-08-13 17:18:16]  Set $GOROOT_BOOTSTRAP to a working Go tree >= Go 1.4.
```
解决办法：

先安装默认版本的go
```
yum install golang
```
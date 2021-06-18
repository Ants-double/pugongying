# ssh无法登录的问题

1. 登录一直被拒绝 使用云服务器控制台进入终端

   

 查看sshd状态

``` shell
systemctl status sshd
```

![error](https://images.cnblogs.com/cnblogs_com/ants_double/1989032/o_21061803281801.png)

判断端口被恶意扫了，尝试修改端口(根本原因就是不让其它恶意用户登录，修改端口以防止他们通过默认端口尝试，其次也可以通过加黑名单或者禁用root用户登录等多种方式，主要目录就是不被发现并用默认端口或帐号多次尝试，以免影响正常使用)

``` shell
vim /etc/ssh/sshd_config
#Port 22
Port 3033
```

然后重启sshd

``` shell
systemctl restart sshd
```

打开防火墙端口

``` shell
firewall-cmd --zone=public --add-port=3033/tcp --permanent
firewall-cmd --reload 
```



2. 在上面启动的过程中报255的错误，查看日志显示  error: Bind to port 3303 on 0.0.0.0 failed: Permission denied

发现是SELinux的问题 关闭SELinux

``` shell
 sudo setenforce 0
 getenforce
 # 输出
 Permissive
 #此命令临时关闭 selinux  重启失效
 sudo vim /etc/sysconfig/selinux

找到

SELINUX=enforcing

替换为

SELINUX=disabled

保存退出

reboot

然后重启ssh访问即可。
```






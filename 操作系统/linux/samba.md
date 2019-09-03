# linux共享目录之samba服务

安装

```
yum -y install samba
```

检查安装情况

```
rpm -qa | grep samba
```

查看安装位置

```
whereis samba
```

修改配置文件：vi /etc/samba/smb.conf, 添加

```
[share]
        comment = Shared Folder with username and password
        path = /data01/magneto/localSharedDir
        valid users = magneto
        public = no
        writable = yes
        printable = no
        create mask = 0765
```

将linux系统已存在用户username加入到Samba用户数据库，windows访问samba共享目录时需要输入此用户名和密码

```
smbpasswd -a magneto
```

启动

```
service smb start
```

常见问题

```
getsebool -a | grep samba // 查看
setsebool -P httpd_enable_homedirs=0 // 关闭打开的
setenforce 0 // 设置SELinux服务与策略，使其允许通过Sambaf服务程序访问普通用户家庭目录, 开放共享目录权限，确保setlinux关闭
```

挂盘

```
sudo mount -t cifs -o username=Administrator,password=Pentest1179 //ip/ ./windows
sudo mount //ip/testData /data01/magneto/windowsSharedDir -o iocharset=utf8,username=administrator,password='Pentest1179',uid=0 -t cifs
```
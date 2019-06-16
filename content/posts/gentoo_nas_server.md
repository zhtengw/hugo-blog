---
title: "使用Gentoo系统作为NAS"
date: 2019-06-16T20:16:18+08:00
draft: false
toc: true
images:
tags: 
  - gentoo
  - linux
  - server
  - NAS
---


相关 **USE Flag** 在此文件中设定：
`/etc/portage/package.use/server`

### Samba 服务[^1][^2]
```shell
# emerge -av samba
# cp /etc/samba/smb.conf.default /etc/samba/smb.conf
# smbpasswd -a aten
New SMB password:
Retype new SMB password:
Added user aten.
```
`/etc/samba/smb.conf`:
```ini
[global]
...
   ; 不指定用户时使用guest模式访问，如果不加这行，每次都会要求认证用户名
   map to guest = bad user 
...

# 用户认证后可以访问个人home目录
[homes]
   comment = Home Directories
   browseable = no
   writable = yes

# 默认guest模式的访问目录
[public]     
   path = /home/aten/public  
   public = yes
   create mask = 0666
   directory mask = 2777
   browseable = yes
   guest ok = yes
   writable = yes
```
```shell
# rc-update add samba default
# rc-service samba start
```

### Webdav 服务
```shell
# echo "www-servers/apache APACHE2_MODULES: dav dav_fs dav_lock" >> /etc/portage/package.use/server
# emerge -av apache
```

`/etc/conf.d/apache2`:
```ini
APACHE2_OPTS="... -D DAV"
```
`/etc/apache2/httpd.conf`:
```ini
...
# 开启服务后，对网页的所有操作都会以下面规定的用户和组来进行，如果/webdav目录的属主不同，则可能无法读写
User aten
Group aten
...
```
`/etc/apache2/modules.d/45_mod_dav.conf`:
```ini
...
DavLockDB "/home/aten/lockdb" # /home/aten must belong to User:Group
Alias /webdav "/home/aten/webdav"
<Directory /home/aten/webdav>
Dav On
Options +Indexes # 如果没有此项，浏览器无法列出文件
AddDefaultCharset UTF-8
AuthType Basic
AuthName "WebDAV Server"
AuthUserFile /etc/apache2/users.pwd
Require valid-user # 只有在users.pwd中授权的用户名才能浏览
                  # 这个用户名可以跟系统的账户不同
</Directory>
...
```
```shell
# su -c "mkdir /home/aten/webdav" - aten
# htpasswd -c /etc/apache2/users.pwd aten
New password: 
Re-type new password: 
Adding password for user aten

# rc-update add apache2 default
# rc-service apache2 start
```
### Apple Time Machine 服务[^3]
```shell
# echo "net-fs/netatalk zeroconf" >> /etc/portage/package.use/server
# emerge -av netatalk

```
`/etc/afp.conf`:
```ini
[TimeMachine]
  path = /home/aten/AppleTimeMachine
  time machine = yes
```

```shell
# su -c "mkdir /home/aten/AppleTimeMachine" - aten
# chmod 777 /home/aten/AppleTimeMachine

# rc-update add netatalk default
# rc-service netatalk restart
```

### 智能家居终端

#### homeassistant
```shell
# eselect repository enable HomeAssistantRepository
# emerge --sync HomeAssistantRepository
# echo "app-misc/homeassistant miio asuswrt" >> /etc/portage/package.use/server
# emerge -av homeassistant
# rc-update add homeassistant default
# rc-service homeassistant start
```
可通过浏览器访问`http://[ip]:8123`打开homeassistant的网页管理
 
### Kodi

### Plex

[^1]: [Create Samba Shares on Ubuntu for Windows Systems](https://websiteforstudents.com/lesson-12-create-samba-shares-ubuntu-windows-systems/)
[^2]: [Samba in Arch Wiki](https://wiki.archlinux.org/index.php/Samba#User_Management)
[^3]: [Netatalk in Gentoo Wiki](https://wiki.gentoo.org/wiki/Netatalk)

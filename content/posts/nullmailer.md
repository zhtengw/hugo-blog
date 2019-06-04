---
title: "设置nullmailer发送邮件"
date: 2019-05-31T16:14:03+08:00
draft: false
toc: true
images:
tags:
  - gentoo 
  - linux
  - mail
  - server
  - cron
---

打算写个脚本，检查linuxdeepin的各个项目的GitHub有无更新，如果有更新便在deepin-overlay自动创建新版本的ebuild，并测试安装，然后把结果发到我邮箱。这便要安装一个命令行的发信系统。Gentoo默认系统已经安装了 **mail-mta/nullmailer** [^1]，我就直接配置它便好了。
### 设定 SMTP 服务器
要成功发信到互联网邮箱，这便要求配置可用的 **SMTP** 服务器。对于nullmailer，这在 **/etc/nullmailer/remotes** 中设定。默认的 remotes 文件中有很详细的注释说明，参考注释很容易就设定好了：

**/etc/nullmailer/remotes**
```ini
smtp.163.com smtp user=myname@163.com pass=myspecialpassword port=465 ssl
```
尝试发件：
```bash
# sendmail 命令会把邮件放到待发送队列
$ echo "Test mail from nullmailer at $(date)" | sendmail recname@mail.domain
# 使用 nullmailer-send 将邮件发送出去
$ sudo nullmailer-send
```

却提示失败：
```
Starting delivery, 1 message(s) in queue.
Starting delivery: host: smtp.163.com protocol: smtp file: 1559199706.31426
From: <aten@gentoo.vm> to: <recname@mail.domain>
Message-Id: <1559199706.945315.31427.nullmailer@me>
smtp: Failed: 553 Mail from must equal authorized user
Sending failed: Permanent error in sending the message
Moving message 1559199706.31426 into failed
Not generating double bounce for 1559199706.31426
Delivery complete, 0 message(s) remain.
```
原因很容易发现，**smtp: Failed: 553 Mail from must equal authorized user** 和 **From: <aten@gentoo.vm>** 提示我们，sendmail 直接用我执行命令的系统**用户名**，加上给系统定义的@**主机名**作为发件人，以至于 SMTP 服务器提示发件人是非认证用户。这要设定 **/etc/nullmailer/allmailfrom** 来更正[^2][^3]：
```bash
$ su -c "echo "myname@163.com" > /etc/nullmailer/allmailfrom"
```
再尝试发送测试邮件，却又被网易认为是垃圾邮件而退回了：
```
Starting delivery: host: smtp.163.com protocol: smtp file: 1559199698.31421
From: <myname@163.com> to: <recname@mail.domain>
Message-Id: <1559199698.551960.31420.nullmailer@me>
smtp: Failed: 554 DT:SPM 163 smtp8,DMCowAD3FCvaf+9cP0pBGA--.58361S2 1559199707,please see http://mail.163.com/help/help_spam_16.htm?ip=113.70.36.158&hostid=smtp8&time=1559199707
Sending failed: Permanent error in sending the message
Moving message 1559199698.31421 into failed
Generating bounce for 1559199698.31421
Delivery complete, 0 message(s) remain.
```
换用 Gmail，重新设定 **/etc/nullmailer/allmailfrom** 和 **/etc/nullmailer/remotes** ，我开启了两步验证，所以这里 Gmail 的密码要用应用程序专用密码：

**/etc/nullmailer/allmailfrom**
```ini
myname@gmail.com
```
**/etc/nullmailer/remotes**
```ini
smtp.gmail.com smtp user=myname@gmail.com pass=appspecialpassword port=587 starttls
```
再次尝试发件，这下成功了。
### 邮件内容编码
我的 Gentoo 系统里设定的字符编码是 UTF-8 的，通常来说没什么问题，可偏偏win版Outlook收中文信件便会乱码。解决办法是，在邮件内容中明确指定编码[^4][^5]：
```bash

# 使用 sendmail 发送
$ echo -e "Subject:测试邮件 \nContent-Type: text/html; charset=utf-8 \n\n Test mail from nullmailer at $(date)\n" | sendmail  recname@mail.domain

# 使用 mail 发送
$ echo "Test mail from nullmailer at $(date)" | sendmail -s "=?UTF-8?B?`echo "Test mail" | base64`?="  recname@mail.domain
```
### 自动发信 
接下来只要启动 nullmailer 服务，并把需要定期执行的任务加入 crontab ，cron 程序便会自动在任务执行完后将其产生的所有输出用邮件发送了。
```bash
$ systemctl enable nullmailer
$ systemctl start nullmailer
$ crontab -e
    MAILTO=recname@mail.domain
    00 4 * * * /PATH/COMMAND
```
可以用输出重定向很方便地决定哪些输出结果需要发邮件：
```bash
# 任何输出都不发送邮件
00 4 * * * /PATH/COMMAND > /dev/null 2>&1
# 只发送出错信息
00 4 * * * /PATH/COMMAND > /dev/null
# 只在出错时发送，内容包含所有输出
00 4 * * * /PATH/COMMAND > log_file 2>&1 || (cat log_file)

```

[^1]: [代轉信件的 Nullmailer](https://ithelp.ithome.com.tw/articles/10133399)
[^2]: http://www.troubleshooters.com/linux/nullmailer/
[^3]: [Why is nullmailer appending my hostname to the recipient address?](https://unix.stackexchange.com/questions/505750/why-is-nullmailer-appending-my-hostname-to-the-recipient-address)
[^4]: [Linux send邮件中文乱码](https://0001111.iteye.com/blog/1539446)
[^5]: [How to send email message content as HTML rather than plain text?](https://superuser.com/questions/243152/how-to-send-email-message-content-as-html-rather-than-plain-text)

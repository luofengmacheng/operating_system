## Linux SSH登录的若干问题

### 1 公钥免密登录

在客户端生成公钥和私钥：

```
ssh-keygen -t rsa
```

生成过程中不需要输入密码，如果输入了密码，在进行验证时还是需要密码验证。

然后，在相应的目录(一般是~/.ssh/)下生成公钥和私钥：id_rsa.pub和id_rsa。

将公钥中的内容拷贝到远程机器的登录用户的~/.ssh/authorized_keys：

* scp直接拷贝，然后重定向到authorized_keys
* ssh-copy-id -i ~/.ssh/id_rsa.pub root@127.0.0.1

### 2 免交互的公钥分发

无论使用scp还是ssh-copy-id进行公钥分发时，都需要输入密码进行验证。要想无交互的公钥分发，就需要将密码放在参数中，可以使用sshpass：

```
sshpass -p 密码 ssh-copy-id -i ~/.ssh/id_rsa.pub "-o StrictHostKeyChecking=no 127.0.0.1"
```

### 3 关于known_hosts文件

当客户端第一次ssh登录某台机器时，客户端会询问是否将计算的公钥保存到~/.ssh/known_hosts中。当下次访问同一台机器时，会核对机器的公钥是否与known_hosts文件中的公钥是否一致，如果不一致，则会报错。

于是这个会带来两个问题：

* 使用sshpass进行公钥分发时还是会需要交互
* 当机器重装后，公钥变了，则无法进行登录

解决的办法：

* 删除客户端中的~/.ssh/known_hosts文件中的该机器的公钥
* 进行ssh连接时加上`-o StrictHostKeyChecking=no`选项，使得不进行公钥的验证
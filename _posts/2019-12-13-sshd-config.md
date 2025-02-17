---
layout: post
title:  "sshd 密钥登录"
date:   2019-12-13
categories: linux
---


自己电脑，有时候会共享数据，当服务器让别的电脑利用 ssh来登录。
sshd 只有在需要的时候才打开。但是这样太麻烦，长期开着 sshd 太
危险。有两种措施可以减小被入侵的概率:  

- 更改 sshd 默认登录端口
- 使用密钥登录，关闭密码登录

这样基本上被入侵的概率就很小了。
两项配置都是通过修改 sshd 配置文件来实现。通常配置文件位于
/etc/sshd/sshd_config。文件里面每一项有详细说明，或者通过 
man sshd_config 来查阅。

### step.1 修改默认端口

```
Port 22 # 改成你想要的
```

### step.2 使用密钥登录

使用密钥登录会用到非对称加密，功能大致为：

1. 公钥是公开的，用来加密
2. 私钥是保密的，用来解密
3. 算然已知公钥，但很难解出私钥

sshd 密钥登录，简单来说就是，客户机生成公钥，提前给到服务器保存，然后服务器才允许公钥已备案的客户端登录。
就像电话预约餐厅一样，必须先知会餐厅，之后才能在餐厅有位置吃饭。

- 生成密钥 ssh-keygen -t rsa
- 拷贝到服务器账户的 ~/.ssh/authorized_keys 文件里
- 修改 sshd 配置项 PubkeyAuthentication 为 yes

### step.3 禁用密码登录
```
PasswordAuthentication no
```

最后重启 sshd: sudo systemctl restart sshd
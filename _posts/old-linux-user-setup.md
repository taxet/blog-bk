---
title: LINUX创建用户没有密码只使用ssh登录
date: 2018-07-02 17:12:34
tags: 
- 运维
- linux
category: 运维
---

1. 添加用户（不要密码）

```bash
useradd -m -d /home/《username》 -s /bin/bash 《username》
```

2. 添加sudo权限

```bash
visudo
```

之后添加下面语句

```bash
《username》 ALL=(ALL) NOPASSWD:ALL
```

或者

```bash
echo '《username》 ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/《username》
```

3. 进入用户，创建密钥

```bash
su 《username》
ssh-keygen
```

4. 处理密钥

```bash
cd ~/.ssh
cat id_rsa.pub >> authorized_keys
```

之后再吧id_rsa拷出来作为秘钥

5. SSH登录设置

* 设置正确的权限

```bash
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

随后，在/etc/ssh/sshd_config里加上

```bash
RSAAuthentication yes
PubkeyAuthentication yes
```

---
title: 我是如何使用docker搭建一个hexo博客的
tags:
  - hexo
  - docker
  - certbot
category: 运维
date: 2019-11-06 11:03:21
---

虽然网上[hexo](https://hexo.io)的教程很多，但是大多数都是教你如何弄到github page上的，很少有教你用docker把hexo部署到容器里的。不过经过几个小时的摸索，我终于自己撸了一遍，先记录下来。

首先是docker-compose.yml文件：

```yml
version: '3'
services:
  hexo:
    container_name: hexo
    image: ipple1986/hexo
    ports:
    - '4000:4000'
    volumes:
    - './_config.yml:/opt/hexo/ipple1986/_config.yml:rw'
    - './themes:/opt/hexo/ipple1986/themes:rw'
    - './source:/opt/hexo/ipple1986/source:rw'
```

说明一下，hexo似乎没有官方的docker hub，所以这里用了星数最高的ipple1986/hexo，非常的原生态
这个hub的所有hexo文件会放在容器里的/opt/hexo/ipple1986路径下面，把里面的_config.yml文件和themes以及source文件夹映射出来就可以了，如果需要的话也可以把其他文件夹也给映射出来，我这里不需要所以就算了。

接下来启动

```bash
docker-compose up -d
```

由于我使用的theme要装一些其他的插件，所以说还需要进到容器里面去装

```bash
docker exec hexo sh -c 'npm install hexo-renderer-sass --save && npm install hexo-renderer-jade --save'
```

最后重启一下docker就大功告成了

[最终效果](https://blog.sayuri.moe)

### 添加feed

首先安装hexo-generator-feed

```bash
docker exec hexo sh -c 'npm install hexo-generator-feed --save'
```

之后再_config.yml里面添加设置

```yml
plugin:
- hexo-generator-feed
feed:
  type: atom
  path: atom.xml
  limit: 20
```

重启容器就可以了，[连接](https://blog.sayuri.moe/atom.xml)

---

然而在弄https的时候又出问题了，一般使用certbot的语句是

```bash
sudo certbot --nginx
```

然而这个时候却跳出来 **No module named 'requests.packages.urllib3'**的错误
去网上找了一下，发现centOs的操作系统对这个的支持都不是很好
心情复杂.jpg

开始解决问题吧

### 首先试试pip强装urllib和request看看行不行

urllib3和pyOpenSSL装的都还好，不过到了requests

```bash
sudo pip install reuests
```

给出提示：

```log
ERROR: Could not find a version that satisfies the requirement reuests (from versions: none)
ERROR: No matching distribution found for reuests
```

之后我去[直接下了一个](https://pypi.org/project/requests/)

```bash
sudo pip install 那个文件.whl
```

装是装上去了， 但是遇到了新的问题

```log
ImportError: cannot import name UnrewindableBodyError
```

网上说是要升级一下urllib3

```bash
sudo pip install --upgrade urllib3
```

然后你猜猜又发生了什么？
当然是报错啦！

```log
AttributeError: 'module' object has no attribute 'pyopenssl'
```

又是urllib3的问题啊……

网上找了一圈都没找到问题所在，换个思路吧

### 直接去下个certbot会如何呢

[这里](https://pypi.org/project/certbot)

```log
No module named requests
```

大失败！

### 之后我寻思这是在阿里云的机器上的问题，直接去搜索阿里云是否有同样糟糕的情况会不会好一点

找到了[这个](https://yq.aliyun.com/articles/713724)和[这个](https://yq.aliyun.com/articles/597943)
似乎也不行

### 试试强制重装requests

```bash
sudo pip install requests --force --upgrade
```

然而会报错：

```log
ERROR: Cannot uninstall 'requests'. It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.
```

很这个报错是红色的，很吓人哦~

原来是需要摆脱依赖进行更新，需要加--ignore-installed参数

```bash
sudo pip install requests --force --upgrade --ignore-installed
```

之后再试，发现报错的姿势变了

```log
AttributeError: 'module' object has no attribute 'pyopenssl'
```

我可是身经百战了，自然知道应该怎么做了

```bash
sudo pip install pyopenssl --force --upgrade --ignore-installed
```

不行

试试下个思路吧

### 去官网看看吧

我直接在certbot的[官网](https://certbot.eff.org/lets-encrypt/centosrhel7-nginx)上找到centOs7的安装方法

然而出现了这样的错误：

```log
DistributionNotFound: The 'certbot==0.39.0' distribution was not found and is required by the application
```

之后我pip freeze后发现没有certbot，于是用pip重新装一遍

```bash
sudo pip install certbot==0.39.0
```

然而

```log
AttributeError: 'module' object has no attribute 'pyopenssl'
```

怎么老是你

之后发现要从yum装而不是pip装

```bash
pip uninstall requests
yum reinstall python-requests

pip uninstall six
yum reinstall python-six

pip uninstall urllib3
yum reinstall python-urllib3
```

解决了，完美

以后我再也不用centOS了！

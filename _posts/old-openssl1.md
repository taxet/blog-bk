---
title: openssl使用相关简介(一)
date: 2018-01-11 18:16:34
tags: 
- 运维
- openssl
category: 运维
---
## openssl使用相关简介

## 简单的加密一个文件

```bash
openssl enc -aes-256-cbc -in input.txt -out output.bin -pass pass:111111
```

解释一下：
* openssl enc: openssl里面的文件加密功能
* -aes-256-cbc: 加密方法为aes-256-cbc，至于全部的加密方法可以输入openssl enc -help查看
* -in input.txt: 输入文件为input.txt
* -out output.bin: 输出文件为output.bin
* -pass pass:111111: 密码为111111

加上-d可以解密
```bin
openssl enc -aes-256-cbc -d -in out.bin -pass pass:111111
```

## 公钥与私钥

非对称算法（例如rsa）都需要一个加密密钥和一个解密密钥，不同的场景下这两个密钥可以分别作为公钥与私钥。

比如说我想每个访问我的人都通过密文来传递信息，则可以使用公钥加密私钥解密（SSH通信）。而我想证明某个东西是我发出来的，则可以通过私钥加密公钥解密（证书）。

下面生成一个简单的rsa密钥对：
```bash
openssl genrsa -out key.pem 1024
```

解释一下
* openssl genrsa: 官方文档上说的是生成rsa私钥，但实际上生成的是rsa密钥对
* -out key.pem 输出文件为key.pem
* 1024: 不是哪个意思，指的是密钥大小为1024个比特，默认为512，一般工程上使用的是1024， 2048或更大

之后提取出公钥（加密密钥）:
```bash
openssl rsa -in key.pem -pubout -out pub-key.pem
```

然后试试加密文件：
```bash
openssl pkeyutl -encrypt -in a.txt -pubin -inkey pub-key.pem -out b.bin
```
解释一下：
* -inkey pub-key.pem 加密密钥为pub-key.pem
* -pubin 指加密密钥只含有公钥，如果不加这个参数则会出现 unable to load Private Key 的错误

同样，解密：
```bash
openssl pkeyutl -decrypt -in b.bin -inkey key.pem -out c.txt
```

## 数字签名
电子签名的作用是
1. 某人发布过某个消息
2. 消息的来源是某人
3. 途中没有丢失或篡改

电子签名的实现一般是由加密密钥做私钥，解密密钥做公钥。

参考步骤：
```bash
openssl dgst -sha512 -out hash.txt a.txt
```

解释一下：
* -sha512 哈希算法为sha512，全部的算法可以通过openssl dgst -help查看
* -out hash.txt 输出文件为hash.txt
* a.txt 需要处理的文件为a.txt

之后使用密钥签名
```bash
openssl pkeyutl -sign -in hash.txt -out a.sig -inkey key.pem
```

验证：
```bash
openssl pkeyutl -verify -in hash.txt -sigfile a.sig -inkey pub-key.pem -pubin
```
不想解释

当然，也可以直接用dgst来加密和验证（推荐）
```bash
#签名
openssl dgst -sha512 -sign key.pem -out a.sig a.txt
#验证
openssl dgst -sha512 -verify pub-key.pem -signature a.sig a.txt
```
解释一下

签名中
* -sign 指签名，后面加私钥文件
验证中
* -verify 指验证，后面加公钥文件
* -signature 后面加签名文件
(未完待续)

### 参考资料
[A 6 Part Introductory OpenSSL Tutorial](https://www.keycdn.com/blog/openssl-tutorial/)
[openssl documentation](https://www.openssl.org/docs/)
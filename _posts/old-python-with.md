---
title: python中的with异常处理与JAVA中类似的比较
date: 2018-01-05 19:23:34
tags: 
- JAVA
- PYTHON
category: CODING
---

一般情况下python的异常处理是用try ... except ... finally来处理的，例如打开一个文件

```python
f = open(filename)
try:
    ''' do something '''
except IOError:
    print('I/O error')
finally:
    f.close()
```

这么做的缺点是非常麻烦，特别是如果里面有嵌套的try ... except的语句的话就等着加游标卡尺吧。所以说python提供了with关键词来解决这个问题，以上的代码可以变成：
```python
with open(filename) as f:
    ''' do something'''
```
变成一句了，是不是觉得变得简单了？
而且还可以处理多个：
```python
with open(file1) as f1, open(file2) as f2:
   ''' do something'''
```
不过使用with会直接将错误抛出，目前还没有找到在with里特殊处理错误的方法。

同样，JAVA7往后的版本也提供了类似的东西：
```java
InputStream i = getStream()
try {
  // do something
} catch(Exception e) {
  e.printStack();
} finally {
  try {
    i.close();
  } catch(Exception e) {
    e.printStack();
  }
}
```
变成
```java
try(IntpuStream i = getStream()) {
  // do something
} catch(Exception e) {
  e.printStack();
}
```
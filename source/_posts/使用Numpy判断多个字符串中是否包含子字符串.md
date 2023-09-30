---
title: 使用Numpy判断多个字符串中是否包含子字符串
date: 2023-09-30 12:42:21
tags:
- Numpy
---

> 这几天在做项目的时候遇到一个问题：需要判断多个字符串中是否包含子字符串，数据量大概在30W左右。使用 for 循环一个一个判断显然是不可行的，Numpy是 for 循环的最好替代者。遂去 Numpy 官方文档看了一下，并找到了解决方案🥰。

# char.count() 函数

```python
numpy.char.count(a, sub, start, end)
```

该函数可以用来统计 sub 在 a 中出现的次数，具体介绍可以看[官方文档](https://numpy.org/devdocs/reference/generated/numpy.char.count.html#numpy.char.count)。



# 对 char.count() 函数进行改造

对该函数稍加修改就能满足我们的需求。修改如下：

```python
numpy.char.count(a, sub, start=0, end=None) != 0
```

经过这样处理之后，返回的就是一个 **bool 数组**，如果 sub 在 a 中出现了就返回 True ，否则返回 False 。



- 举例：

```python
import numpy as np
a = np.array(['hello world', 'hello python'], dtype='str')
np.char.count(a, 'py')
# array([0, 1])
np.char.count(a, 'py') != 0
# array([False, True])
```


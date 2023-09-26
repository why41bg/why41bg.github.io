---
title: 对使用ajax技术的网页的爬取实现
date: 2023-09-26 11:07:23
tags:
- ajax
- 爬虫
---

> 这几天在使用 `Python爬虫` 爬取东方财富网的数据中心，但是老师要求只能使用 `BeautifulSoup` 和  `Xpath` 这两个库（禁止使用 `selenium` 库）。在仔细分析东方财富网的网页结构和所用技术之后，我发现仅仅依靠这两个库是完全不可能实现的。下面细嗦。



# 网页HTML分析🥴

一开始，直接使用 `requests` 库向网页url发GET请求拿到的网页源码中没有找到任何要爬取的目标数据，初步判断该网页是一个动态网页，数据来源于后端。在浏览器中打开开发者选项对该网页进行抓包，发现在打开网页和翻页的时候确实存在向另外一个不同的网址发送GET请求的包。这应该就是后端的地址了。😮



# 所用技术分析😵

对网页前端向后端发送的包进行分析。不难看出是使用ajax发送的GET请求包。

而网站的翻页，是使用的js实现。



# 可行办法分析😴

由于规定了实现的途径（禁止模拟），我又不想js逆向，便只好采用直接向后端发请求拿数据的办法了。这样一来，甚至连 `BeautifulSoup` 和 `Xpath` 库都用不上了。😢~~我是真搞不懂老师为什么要这么规定~~



# 实现😬

- 关键代码如下：

```python
url = 'https://datacenter-web.eastmoney.com/api/data/v1/get'  # 后端地址
headers = {
    'Accept': '*/*',
    'Accept-Encoding': 'gzip, deflate, br',
    'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6',
    'Connection': 'keep-alive',
    'Cookie': '',  # 此处填入自己的cookie
    'Dnt': '1',
    'Host': 'datacenter-web.eastmoney.com',
    'Referer': 'https://data.eastmoney.com/yjfp/',
    'Sec-Ch-Ua': '"Microsoft Edge";v="117", "Not;A=Brand";v="8", "Chromium";v="117"',
    'Sec-Ch-Ua-Mobile': '?0',
    'Sec-Ch-Ua-Platform': '"Windows"',
    'Sec-Fetch-Dest': 'script',
    'Sec-Fetch-Mode': 'no-cors',
    'Sec-Fetch-Site': 'same-site',
    'User-Agent': ''  # 填入
}
params = {
    # 'callback': 'jQuery11~~~',  # 这行参可以不要，直接注释掉即可
    'sortColumns': 'PLAN_NOTICE_DATE',
    'sortTypes': '-1',
    'pageSize': '5000',  # 一次返回多少数据，这里设置大一点就可以一次性拿完全部数据，不用翻页操作了，翻页操作其实也就是再发请求而已
    'pageNumber': '1',
    'reportName': 'RPT_SHAREBONUS_DET',
    'columns': 'ALL',
    'quoteColumns': '',
    'js': '{"data":(x),"pages":(tp)}',
    'source': 'WEB',
    'client': 'WEB',
    'filter': f"(REPORT_DATE='2023-6-30')",  # 表格上方选择框的分红送配报告期
}

# 填好参数后，直接发送请求即可拿到数据
response = requests.get(url=url, headers=headers, params=params)
response = response.json()  # 将数据转为json格式
```



# 结论😮‍💨

对于静态网页，传统爬虫模式绰绰有余。

对于动态网页，如果嫌弃selenium太慢，就老老实实抓包找出后端地址吧。🧐

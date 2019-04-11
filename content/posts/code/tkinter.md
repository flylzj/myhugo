---
title: "Tkinter"
date: 2018-11-16T17:12:54+08:00
showDate: true
draft: false
tags: ["python"]
---

# tkinter学习总结

最近用gui写一个电影订票系统, 之前也写过一个爬虫的，不过那个主要逻辑在爬虫，这次完了下所有逻辑都在页面的qaq，用的模式是MVC。

## 开始

关于tkinter，tkinter模块是实现了tcl/tk接口的一个模块，也是python自带的gui模块。
tkinter的用法,在我看来就是创建若干个主窗口,然后加上若干个小组件来实现想要的效果。

## tkinter的基本组件

tkinter的源码里有个BaseWidget()类,这个是所有部件的父类.其他部件都在他的基础上实现的.

聊一下关于这些部件的使用(部分使用过的).

### Toplevel

Toplevel相当于一个二级窗口,一般用在开启一个子窗口.

在这个项目里我在选座和添加电影功能里都使用了,代码里的`create_add_movie_top_level`方法通过一个按钮来调用,创建一个toplevel,然后在添加

```python
    def create_add_movie_top_level(self):
        self.movie_top_level.update()
        # 这里是因为这个toplevel的创建是在类的`__init__`方法,
        # 但是如果不隐藏的话在这个类被创建后toplevel也会被创建,
        # 所以在`__init__`方法里我加了self.movie_top_levle = tk.Toplevel(self),self.movie_top_level.withdraw()来暂时隐藏,
        # deiconify方法是取消隐藏
        self.movie_top_level.deiconify() 
        self.movie_top_level.wm_attributes("-topmost", 1)
        tk.Label(self.movie_top_level, text='添加电影').grid(row=0, column=1)
        tk.Label(self.movie_top_level, text='电影名：').grid(row=1, column=1)
        tk.Entry(self.movie_top_level, textvariable=self.added_movie_name).grid(row=1, column=2)
        # tk.Label(movie_top_level).grid(row=2)
        tk.Label(self.movie_top_level, text='简介：').grid(row=3, column=1)
        tk.Entry(self.movie_top_level, textvariable=self.added_movie_desc).grid(row=3, column=2)
        # tk.Label(movie_top_level).grid(row=4)
        tk.Label(self.movie_top_level, text='放映时间：').grid(row=5, column=1)
        self.create_on_show_time_frame(self.movie_top_level).grid(row=5, column=2)
        self.add_movie_button.grid(row=6, column=2)
```

效果是这样的
![toplevel](../tkinter-toplevel.png)

### Button

button就是按钮,应该很好理解.
定义的话直接
```
button = tk.Button(master=None)
```
他有一些常用的参数,比如master,上一级widget,默认是None,为None的话就是root.text,按钮里的文字.textvariable,绑定tk变量的值.

### Checkbutton

checkbutton,选择按钮,用来添加选择选项.

在这个项目里我的选座就是通过这个实现的,这是代码段

```python
        for seat, is_sche in seats.items():
            v = tk.IntVar(self.window, name='seat{}'.format(seat))
            cb = tk.Checkbutton(self.seat_top_level, text=str(seat), variable=v)
            cb.grid(row=1, column=seat)
            if is_sche:
                cb.select()
                cb.configure(state=tk.DISABLED)
            else:
                cbs.append((v, cb))
```
这里用for循环来创建一定数量的checkbutton,is_sche是一个Boolean类型,如果为true,checkbutton就变成已选并且不可选状态.

效果如下:
![checkbutton](../tkinter-checkbutton.png)

### Entry

entry, Entry widget which allows displaying simple text,源码里的解释就是这个.其实就是一个用来展示和输入文本的地方.

最常见的就是用户名密码的输入.
```python
    def create(self):
        tk.Label(self).grid(row=0, stick=tk.W, pady=10)
        tk.Label(self, text='用户名: ').grid(row=1, stick=tk.W, pady=10)
        tk.Entry(self, textvariable=self.root.username).grid(row=1, column=1, stick=tk.E)
        tk.Label(self, text='密码: ').grid(row=2, stick=tk.W, pady=10)
        tk.Entry(self, textvariable=self.root.password, show='*').grid(row=2, column=1, stick=tk.E)
        tk.Label(self, text="登录角色").grid(row=3, column=0, stick=tk.W)
        tk.Radiobutton(self, text="用户", variable=self.root.login_role, value='user').grid(row=3, column=1)
        tk.Radiobutton(self, text="员工", variable=self.root.login_role, value='staff').grid(row=3, column=2)
        self.login_button = tk.Button(self, text='登录')
        self.login_button.grid(row=5, stick=tk.W, pady=10)
        self.to_register_button = tk.Button(self, text="注册")
        self.to_register_button.grid(row=5, stick=tk.E, column=2)
```

通过绑定StringVar()来获取用户输入.

### Frame

frame在我的理解就是一个装其它部件的容器,通过frame可以很容易的把代码解耦.

```python
class MainFrame(tk.Frame):
    def __init__(self, root):
        super().__init__()
        self.root = root
        self.main_notebook = ttk.Notebook(root)
        self.user_info_frame = UserInfoFrame(root)
        self.movie_frame = MovieFrame(root)
        self.order_frame = OrderHistoryFrame(root)
        self.movie_manager_frame = MovieManagerFrame(root)

    def create(self, role):
        if role == 'user':
            self.main_notebook.add(self.movie_frame, text='首页')
            self.main_notebook.add(self.order_frame, text='订单')
        else:
            self.main_notebook.add(self.movie_manager_frame, text='电影管理')
        self.main_notebook.add(self.user_info_frame, text='我的')
        self.main_notebook.pack()
```

在这个项目里登录之后会展示一个main_frame,他的构造函数里有很多小的frame,他们各自分离,互相不影响.子frame又可以定义子frame.

源码里还说can have a 3D border.但是没用过qaq

### Label

label就是一个放文本或者图片的东西(虽然没放过图片).

### Menu

menu就是普通的菜单.感觉菜单可以看成装了很多button的一个容器吧.

在这个项目里我在订单历史中添加了右键删除订单还有选座的时候右键选座.通过add_command函数添加command到菜单里
```python
        menu = tk.Menu()
        menu.add_command(label='删除', command=lambda: self.delete_item(item, 'history'))
        menu.post(event.x_root, event.y_root)
```

效果如下(好丑哇:

![menu](../tkinter-menu.png)

### Radiobutton

多选一

代码
```python
        tk.Radiobutton(self, text="用户", variable=self.root.login_role, value='user').grid(row=3, column=1)
        tk.Radiobutton(self, text="员工", variable=self.root.login_role, value='staff').grid(row=3, column=2)
```

效果:
![radiobutton](../tkinter-radiobutton.png)

## 还有一些ttk里面的组件

### Treeview

Treeview相当于是创建一个表格,由表头和列组成,每一行都算作是一个item,通过tree.item(item)来访问.通过tree.insert()来插入行

项目里查看正在上映的电影,订单都是用Treeview来实现的

```python
    def create(self):
            self.movies_tree = ttk.Treeview(self, show="headings", columns=('on_show_id',
                                                                        'movie_name',
                                                                        'desc',
                                                                        'on_show_time',
                                                                        'now_seats',
                                                                        'all_seats'))
        self.movies_tree.grid(row=0)
        self.movies_tree.column('on_show_id', width=50, anchor='center')
        self.movies_tree.heading('on_show_id', text='场次id')
        self.movies_tree.column('movie_name', width=100, anchor='center')
        self.movies_tree.heading('movie_name', text='电影名')
        self.movies_tree.column('desc', width=100, anchor='center')
        self.movies_tree.heading('desc', text='简介')
        self.movies_tree.column('on_show_time', width=200, anchor='center')
        self.movies_tree.heading('on_show_time', text='开场时间')
        self.movies_tree.column('now_seats', width=100, anchor='center')
        self.movies_tree.heading('now_seats', text='已经预定数量')
        self.movies_tree.column('all_seats', width=100, anchor='center')
        self.movies_tree.heading('all_seats', text='座位总数')
```

效果:
![treeview](../tkinter-treeview.png)

## 部件装填

部件通过grid， pack来实现布局管理，他们各有个的有点和缺点，因为不追求布局效果而且对ui硬伤，没怎么用上，这里用网上总结的。

### pack

pack给我的使用体验就是把部件一个一个堆到窗口上去，

* fill 控件填充方式, 参数可以是tkinter.X和tkinter.Y,分别表示在X，Y方向和父窗口填充

* padx 水平外部填充,参数是int

* pady 垂直外部填充

* ipadx 水平内部填充

* ipady 垂直内部填充

* side 按顺序填充,参数可以是 tkinter.LEFT, tkinter.RIGHT, tkinter.CENTER

### grid

grid给我的使用体验就是把窗口分成很多格子，然后把部件放到对应的格子

* row 决定放在哪一行
* column 决定放在哪一列

## 说说关于这个小项目的MVC模式

这个项目的大致就是使用MVC模式:
view包下面的都是定义的窗口类，还有一些tk.StringVar()， tk.IntVar()。这些数据是和controller交互的媒介。
table包下面定义的是sqlalchemy所需要的表类。
model包下面定义的是操作数据的一些类。
controller包下的controller类就是入口类了，它的构造函数里实例化了所有的view和model类。
通过getvar（）方法从窗口拿到数据，然后把数据传给model类后返回结果，显示在view类中。

## 最后

代码虽然说是MVC，但是因为开始写得时候对tk的一些部件不是很熟，导致有点乱（我现在也不想改了qaq），因为tk我写的时候也不是很熟练，写完算是对tk也有了比较多的了解。
太丑了也不想贴出来qaq~
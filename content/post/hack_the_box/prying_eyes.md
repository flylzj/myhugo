---
title: "Hack The Box——Prying Eyes——ImageMagick任意文件读取及imagemagick-convert参数注入"
date: 2023-05-29T15:55:09+08:00
draft: false
---


## 0x00 前言

ciscn失利，打算恶补一些ctf题来缓一缓。

## 0x01 进入正题

首先先进入环境，下载作者给的代码，边访问网站边看代码

看起来应该是一个简单的论坛系统

![1.png](1.png)

核心的api有
```
登录 POST /auth/login
注册 POST /auth/register 
创建/回复帖子 POST /forum/post
```

## 0x02 代码审计

审了一圈代码，唯一可操作的地方应该就是创建帖子(`POST /forum/post`)那一块了，这里有个文件图片上传。

```js
  ValidationMiddleware("post", "/forum"),
  async function (req, res) {
    const { title, message, parentId, ...convertParams } = req.body;
    ...
    let attachedImage = null;
    if (req.files && req.files.image) {
      const fileName = randomBytes(16).toString("hex");
      const filePath = path.join(__dirname, "..", "uploads", fileName);
      try {
        const processedImage = await convert({
          ...convertParams,
          srcData: req.files.image.data,
          format: "AVIF",
        });

        await fs.writeFile(filePath, processedImage);

        attachedImage = `/uploads/${fileName}`;
      } catch (error) {
        req.flashError("There was an issue processing your image, please try again.");
        console.error("Error occured while processing image:", error);
        return res.redirect("/forum");
      }
    }
    ...
  }
```

结合POST表单和代码可以看出来，除了`title, message, parentId`三个参数外，其他参数都被写到convertParams里了，
`const processedImage = await convert`负责用convertParams里的参数对图片进行一个转换。


![2.png](2.png)


看了一下这个convert是来自npm包[`imagemagick-convert`](https://www.npmjs.com/package/imagemagick-convert)的，
而它又是一个[`imagemagick`](https://imagemagick.org/)的简单的命令行调用工具。

那么这题的思路应该是去找`imagemagick`相关的漏洞了。

## 0x03 CVE-2022-44268

查了一下`imagemagick`相关的漏洞，果然有一个高度契合的，CVE-2022-44268 任意文件读取。原理大概就是在恶意的PNG内嵌入一个profile关键字，然后值设置为某个文件名，那ImageMagick会对文件进行读取并写到生成的PNG内，详细可以看[https://www.metabaseq.com/imagemagick-zero-days/](https://www.metabaseq.com/imagemagick-zero-days/)

## 0x04 如何利用

这个漏洞只有转PNG文件的时候才会出现，但是从代码可以看出来作者设置的输出的图片类型是AVIF。

```js
        const processedImage = await convert({
          ...convertParams,
          srcData: req.files.image.data,
          format: "AVIF", // AVIF格式
        });
```

最开始想到的就是参数覆盖，因为convertParams，也就是表单除了`title, message, parentId`以外的其它参数都在convertParams里，
我们可以在表单里传一个`format: "PNG"`，但是js的特性，后面同名的键会覆盖前面的键，因为代码里传入的format在convertParams后面，所以不管format传什么，最后的值都是`AVIF`。


这里又是一个卡住的点，既然问题出在`imagemagick-convert`,那么就去看看`imagemagick-convert`的源码。

## 0x05 imagemagick-convert 参数注入

简单看了一下`imagemagick-convert`的代码，发现确实写的很简单，本质上就是通过`spawn`调用`ImageMagick`的`convert`命令，然后在调用之前进行一些简单的参数拼接。这个`composeCommand`函数就负责对一些预设的`attributesMap`参数进行参数组合。最后再进行一个`spawn`的调用。

```js
    attributesMap = new Set([
        'density',
        'background',
        'gravity',
        'quality',
        'blur',
        'rotate',
        'flip'
    ]);
    composeCommand(origin, result) {
        const cmd = [],
            resize = this.resizeFactory();
        for (const attribute of attributesMap) {
            const value = this.options.get(attribute);
            // 拼接命令
            if (value || value === 0) cmd.push(typeof value === 'boolean' ? `-${attribute}` : `-${attribute} ${value}`);
        }
        if (resize) cmd.push(resize);
        cmd.push(origin);
        cmd.push(result);
        return cmd.join(' ').split(' ');
    }

    const origin = this.createOccurrence(this.options.get('srcFormat')),
        result = this.createOccurrence(this.options.get('format')),
        cmd = this.composeCommand(origin, result),
        cp = spawn('convert', cmd),
        store = [];
```

虽然它用的spawn，我们没办法进行node的命令注入，但是对它拼接的参数，还是有可操作性的。正常情况下，`composeCommand`返回的值是

```json
[
  '-density',    '600',
  '-background', 'none',
  '-gravity',    'Center',
  '-quality',    '75',
  'PNG:-',       'AVIF:-'
]
```

一个参数对应一个值，假如我们在参数的值后面加上别的参数名和参数值呢？比如`backgroud=none -resize 50%`，那cmd就变成了
```json
cmd [
  '-density',    '600',
  '-background', 'none',
  '-resize',     '50%',
  '-gravity',    'Center',
  '-quality',    '75',
  'PNG:-',       'AVIF:-'
]
```

之所以会这样，是因为cmd最后返回的时候是`cmd.join(' ').split(' ')`，相当于把参数重新按空格分割了一下，我们传入的resize参数也就被注入进去了。

## 0x06 最终利用

既然可以注入参数，那我们就不用被js这层壳限制了，`imagemagick`原生的所有参数我们都可以用了，翻了一下[`ImageMagick`](https://imagemagick.org/script/convert.php)的文档，发现有一个`-write filename`，可以把当前状态的图片写进文件里。因为`ImageMagick`这个CVE只要求写入的时候是PNG，所以也是完美符合我们的需求。注入一个`-write exp.png`，就可以成功利用这个CVE了。


那么最终的利用步骤为

1. 用[`poc`](https://github.com/vulhub/vulhub/blob/master/imagemagick/CVE-2022-44268/poc.py)进行恶意图片的生成
`./poc.py generate -o poc.png -r flag.txt`
2. 发送包
![3.png](3.png)
3. 访问`/uploads/exp.png`下载我们写入的图片
4. 执行`./poc.py parse -i exp.png`读取带出来的文件
![4.png](4.png)
5. hex转str之后拿到flag
![5.png](5.png)



## 0x07 总结

这道题的这两个点还是比较有趣的，一个是对现有漏洞的利用，另一个是对冷门开源库（imagemagick-convert这个库npm每周下载量才几百）的审计。

## 0x08 参考链接

1. [imagemagick-convert](https://www.npmjs.com/package/imagemagick-convert)

2. [Annotated List of Command-line Options](https://imagemagick.org/script/command-line-options.php#write)

3. [ImageMagick: The hidden vulnerability behind your online images](https://www.metabaseq.com/imagemagick-zero-days/)

4. [ImageMagick任意文件读取漏洞(CVE-2022-44268)](https://zeo.cool/2023/02/12/ImageMagick%E4%BB%BB%E6%84%8F%E6%96%87%E4%BB%B6%E8%AF%BB%E5%8F%96%E6%BC%8F%E6%B4%9E(CVE-2022-44268)/)

5. [https://github.com/vulhub/vulhub/blob/master/imagemagick/CVE-2022-44268/poc.py](https://github.com/vulhub/vulhub/blob/master/imagemagick/CVE-2022-44268/poc.py)




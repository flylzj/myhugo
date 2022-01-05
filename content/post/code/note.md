---
title: "Note"
date: 2020-08-10T11:21:57+08:00
draft: true
---

# linux命令大全

## 文件管理

### `cat`

#### 作用

连接文件或者标准输入到标准输出

#### 用法

```shell
cat [OPTION]... [FILE]...
```

#### 参数

```shell
-A  --show-all          和-vET等价
-b  --number-nonblank   为非空行打上数字行号
-e                      和-vE等价
-E  --show-ends         在每行结尾显示`$`
-n  --number            为所有行打上数字行号
-s  --squeeze-blank     多个空行连续出现时只输出一个空行
-t                      和-vT等价
-T  --show-tabs         将TAB显示为^T
-u                      （忽略）
-v  --show-nonprinting  显示不可打印字符
```


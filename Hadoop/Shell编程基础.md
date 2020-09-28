# Shell编程基础



### 1. 基本语法

使用vi编辑器新建一个文件hello.sh

```sh
#!/bin/bash
echo "Hello World !"
```

---

**执行方法**

```sh
sh hello.sh
chmod +x ./hello.sh

```



### 2. 变量

局部变量

```shell
#! /bin/bash
str="hello"
echo ${str}world
```

环境变量

```sh
echo $PATH
echo $HOME
env #查看所有的环境变量

vim /etc/profile #在这里面可以修改环境变量
source /ect/profile #使修改的环境变量生效

```



### 3. 特殊字符
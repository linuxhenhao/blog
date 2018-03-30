+++
title = "基于django+redis+mysql的静态网站评论系统"
date = 2018-03-17T17:29:54+08:00
tags = ["Django", "Docker"]
categories = ['web', 'linux']
keywords = ['Django','mysql', 'redis']
draft = true
markup = "mmark"
+++
不知道什么时候流行起来的静态博客风，用github或者gitlab的pages服务搭建静态网站，用markdown
编写网站内容。这样的好处是不需要搭个服务器或者弄个虚拟空间，写起来也很方便。但是缺乏和读者
的互动（如果会有的话[微笑])，原来由网易的评论系统，还有言说等，但是后来或许是为了规避法律
风险，这些服务都停了，所以有了写一个给静态站用的评论系统的想法。
<!--more-->
## 1. 搭建docker开发环境
使用ubuntu的docker镜像作为整个运行环境基础，进行环境的安装配置
1. 修改`/etc/apt/sources.list`，将镜像地址修改为中科大[ustc][2]；网速快的话可以不改
2. 安装相应软件包
```bash
    docker run --name comments -ti ubuntu /bin/bash
    # 启动一个docker container实例，进入alpine的docker环境中

    # 在container实例中运行以下命令
    apt-get update
    apt-get install mysql-server redis-server python3-django
    apt-get install python3-pip  # 用来安装python package
    apt-get install vim
    apt-get install openssh-server  # 添加ssh server，用来外部连接访问
    vi /etc/ssh/sshd_config
    # 修改配置， 将 PermitRootLogin 选项改为 yes，允许root登陆
    passwd  # 设置root的密码，用于ssh连接
    cat > /bin/start << EOF
    #/bin/bash
    service mysql start &  # 后台运行，使得后续命令能够继续下去
    service redis-server start &
    service ssh start &
    while sleep 60;
    do
        echo ""
    done
    EOF
    # 编写启动脚本，用于启动服务，脚本不能退出，否则container就停止运行了
    exit  # 退出container

    # 退出docker环境，回到系统
    docker commit comments comments
    # 将安装配置好的container commit成image
    mkdir comments  # 在host中创建用来保存文件的文件夹
    mkdir -p comments/mysql comments/redis comments/home
    export P=$(pwd)  # 将当前文件夹位置赋值到环境变量P
    docker run --name comments-server --ip=172.17.0.2 \
    -v $P/comments/mysql:/var/lib/mysql -v $P/comments/redis:/var/lib/redis \
    -v $P/comments/home:/root --detach=true comments-server /bin/start
    # 启动服务，可以通过ip连接了，ip地址指定为172.17.0.2，通过在host中运行ifconfig
    # 的到bridge模式下的bridge的ip地址为172.17.0.1，指定为同一网段的任一地址均可
```
3. 修复mysql，通过上述docker run后台启动后，通过ssh连入container，发现mysql并未运行
原因是通过apt-get安装mysql时安装脚本进行了一些配置，生成了文件，位于/var/lib/mysql文件
夹中，使用`-v`指定之后，变成了空文件夹。需要通过ssh连接进行：

     1）将映射后的mysql文件夹的owner:group-owner变成mysql:mysql   
     2）运行`mysqld --initialize --user=mysql`重新初始化一下用户表   
     3）运行`mysqld --skip-grant-tables`，用无验证方式启动一下mysql   
     4）`mysql -u root mysql`连接上无验证方式启动的mysql server，并使用mysql数据库。
`ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码'`，更
新root的密码，之后就可以正常启动，正常使用了。

4. 在`/root`目录中运行`django-admin startproject comments`创建新项目，该文件夹已经被
映射到host的comments/home文件夹，这样就可以在host中写代码了！

5. 修改配置，使用`django_redis`作为缓存，在comments/settings.py中添加
```python
    CACHES = {
        "default": {
            "BACKEND": "django_redis.cache.RedisCache",
            "LOCATION": "redis://127.0.0.1:6379/1",
            "OPTIONS": {
                "CLIENT_CLASS": "django_redis.client.DefaultClient"
                }
            }
        }

    # session也用缓存，这样就会自动使用redis来缓存
    SESSION_ENGINE = "django.contrib.sessions.backend.cache"
    SESSION_CACHE_ALIAS = "default"
```
6. 编辑settings.py配置使用mysql, 需要使用pip安装mysqlclient
```python
    DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'comments',
        'USER': 'username',
        'PASSWORD': 'xxxx',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'OPTIONS': {
            'init_command': 'SET default_storage_engine=INNODB',
        },
    }
    }
```
安装mysqlcient的driver时遇到了mysql_config not found的错误, 需要安装`libmysqlclient-dev`
的包, 如果是其他发行版, 保证所需的mysql_config命令在path中即可. 之后继续pip装driver, 发现
还需要安装gcc, 所以为了正常连接到mysql数据库, 执行以下命令:

    apt-get install libmysqlclient-dev  # 安装后有 mysql_config 命令
    apt-get install gcc python3-dev  # 用于编译mysqlclient, dev包提供头文件
    pip install mysqlclient

django的migrate只会创建表, 而不会创建数据库, 因此需要提前连接创建

    mysql -u root -p  # 用密码登陆
    mysql> create database comments;  # 创建comments数据库(settings.py中指定的)

7. 创建api的app，用RESTFulAPI实现评论的接口`python3 manage.py startapp comments_api`

## 2. 表设计和页面设计
EncryptedPair: 和django的auth的User表进行oneTooneField，内容是根据用户名、密码和salt
生成的appkey和appid，在网站上通过写入appkey和appid来进行权限验证，避免网页中出现明文的
用户名、密码。
Site：和User是1对多关系，用ForeignKey添加Site到

reverse: 在views.py的module level和def xx_view()内部使用reverse时, 只要在module level
用了reverse, 就会导致resolver内部出错, 原因是*reverse需要在urlpatterns定义完了之后反查,
获取到相应的url, 而urlpatterns定义又需要reverse的结果先出来, 即能确定views的内容, 导致
死锁, 这里就需要用到reverse_lazy了.


















[1]: https://alpinelinux.org/
[2]: https://mirrors.ustc.edu.cn/

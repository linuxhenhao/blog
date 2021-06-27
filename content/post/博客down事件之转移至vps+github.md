+++
title = "博客down事件之转移至vps+github"
date = 2018-03-31T10:41:16+08:00
tags = ["blog", "git hooks"]
categories = ['web']
keywords = ['blog','gitlab', 'git hooks']
markup = "mmark"
+++
昨天晚上更新了一篇博客，想看看显示效果，结果发现 gitlab 给了个404，尴尬。过两天是不是可以看
到“你都遇到过什么奇葩简历”的问题下面会有这样的回复“简历上特地标明了有技术博客，然后博客打不开”。
原因gitlab变政策，域名要添加验证txt解析，我没看到，然后就不给用了。决定转到VPS和github算了。
<!--more-->
## 1. 博客404原因
打开 ss 查了下 gmail 的邮件，原来3月23 gitlab 就已经发邮件通知了，域名验证失败，不再对
本博客域名进行转发。具体的说是 gitlab 为了确保 pages 使用者是域名的拥有者，需要添加一条
txt 解析记录。我对这点有疑问，无论使用者是否是域名的拥有着，只要域名拥有着本身同意将域名
CNAME 到 gitlab 的 pages 的地址，有什么问题吗？ 可能只是因为使用者太多，故意添加一条障碍
减少免费用户罢了吧。

从中学到 1. 网站要做好监控 2. 接受监控信息的邮箱不要用需要开SS才能收到的gmail

## 2. 转到VPS+github
既然不爽 gitlab 这么搞事情，那么就把博客放到自己的VPS上去吧。实现的方案是，建一个git repo，
用来存储博客，添加一个 worktree， 然后用 hugo 生成网站内容，使用 nginx 服务。添加一个
post-update 的 hook， 通过密钥验证直接将内容 push 一份到 github 上。

具体操作：

```bash
  # 用 > 表示 root 用户 shell， $ 表示普通用户的 shell
  > useradd git
  > mkdir -p /home/git/blog
  > mkdir /home/git/.ssh
  > cp ~/.ssh/authorized_keys /home/git/  # 使用密钥验证，无需设置密码
  > chown -R git:git /home/git
  > su git
  $ cd /home/git/blog
  $ git init --bare  # 创建一个bare的仓库
  $ cd hooks # 准备设置post-update的hook，该hook会在repo接受push后执行
  $ cat > post-update << EOF  
  #!/bin/sh
  git push  # push 内容到github
  update-blog-public  # 运行 update-blog-public 命令
  EOF
  $ git worktree add ../blogpages  # 添加一个 worktree，用来给 hugo 和 nginx 使用
                                   # 该动作会自动添加一个名为 blogpages 的　branch
                                   # 并且　checkout 到　../blogpages　文件夹
                                   # 每次push后 worktree 中的内容通过 update-blog-public
                                   # 进行更新
  $ ssh-keygen  # 生成不需要密码的sshkey pair，默认名称即可
                # 需要将生成的 /home/git/.ssh/id_rsa.pub 内容添加到 github 账户下
                # 同时 github 上添加一个名为 blog 的空repo
  $ git push -u git@github.com:USER_NAME/blog  # 将 upstream 设置为 github 的repo
  $ exit  # 退出 git 用户，回到 root
  > cat > /usr/bin/update-blog-public << EOF
  #!/bin/sh
  cd /home/git/blogpages
  unset GIT_DIR  # git 优先使用　GIT_DIR 环境变量而非 PWD 作为 repo 的位置
                     # 为了使 cd 生效， unset 掉环境变量
  git merge master  # 更新到 master 的内容
  hugo
  EOF
  > chmod +x /usr/bin/update-blog-public
```
然后在nginx中添加对 /home/git/blogpages/public 作为根文件夹的 server 段就可以了， 注意
添加域名区分之前存在的服务。这样在 VPS 上就实现了 接受 git push -> push 到 github ->
更新 worktree 内容　-> hugo 生成静态页面　-> nginx服务　的功能路线．

其中的　unset GIT_DIR 变量开始没有发现，在 shell 中执行时由于没有 GIT_DIR 这个环境变量，
因此没有出现问题， 但是通过 git push 触发 hooks 时，环境变量中有这个值， 从而导致
`fatal: Not a git repository: '.'` 问题。从 stackoverflow 上找到的答案。

回到个人电脑上：
```bash
  $ cd blog  # 进入blog内容所在的repo
  $ git remote add vps git@xxx.xxx:~/blog  # 添加 vps 上的仓库作远程
  $ git push -u vps  # push 内容到 vps
```
之后，每次更新内容后 git push 就可以了，如果确定不再需要将内容 push 到 gitlab　
也可以将 gitlab 的 remote 删除．

更新记录： 2018-06-11： update-blog-public中的内容修复

title: Hexo+nexT的使用
author: 江小渔
tags:
  - Hexo
categories: []
date: 2019-06-26 08:16:00
---
# Hexo部署

参考官方文档 https://hexo.io/zh-cn/docs/

#### 前提

安装Hexo之前，必须保证自己的电脑中已经安装好了Node.js和Git。因为这两个软件我之前都安装过，这里就不重复安装过程了，检验方式如下：
![upload successful](/images/pasted-0.png)

#### 安装Hexo
```
npm install -g hexo-cli
```
#### hexo-admin
```
npm i hexo-admin --save
```
- 登录`http://localhost:4000/admin`即可看到文章内容，并且在可视化界面中操作文章内容

# 常用指令

```
hexo g #完整命令为hexo generate,用于生成静态文件
hexo s #完整命令为hexo server,用于启动服务器，主要用来本地预览
hexo d #完整命令为hexo deploy,用于将本地文件发布到github上
hexo n #完整命令为hexo new,用于新建一篇文章
```



###### 相关文章
- https://yq.aliyun.com/articles/117271
- https://www.jianshu.com/p/21c94eb7bcd1
- https://www.cnblogs.com/codehome/p/8428738.html
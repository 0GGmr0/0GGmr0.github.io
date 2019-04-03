---
title: hexo + tranquilpeak博客采坑记录  
date: 2019-04-03 14:18:14
tags: [Travis CI]
categories: [博客]
mathjax: false
---
兜兜转转还是决定把博客搭起来，之前那个博客被git吞掉了=。=，只能重新再搭一个，主题选择了看起来很朴素的[tranquilpeak](https://github.com/LouisBarranqueiro/hexo-theme-tranquilpeak)，中间不可避免的踩了不少坑，这里做一个记录。
<!--more-->
## 1 建站
搭建博客的流程如下（先到官网[hexo官网](<https://hexo.io/zh-cn/docs/setup>)安装好需要的组件）

```shell
$ hexo init <folder>
$ cd <folder>
$ npm install
```
删除掉自带的`themes`下的`landscape`主题，从[tranquilpeak relase](https://github.com/LouisBarranqueiro/hexo-theme-tranquilpeak/releases)下载主题zip，解压后重命名为`tranquilpeak`放入`themes`文件夹下,并修改根目录下`_config.yml`文件，修改如下：
```yml
theme: tranquilpeak
```
之后在博客根目录下执行如下命令即可跑起博客
```shell
$ hexo s
```
一些主题的配置参考自[tranquilpeak主题的配置](http://blog.geekap.com/2017/01/07/tranquilpeak%E4%B8%BB%E9%A2%98%E7%9A%84%E9%85%8D%E7%BD%AE/)  
## 2 主题配置的坑
配置`tranquilpeak`主题的时候踩了几个坑，这里记录下
### 2.1 数学公式无法渲染
解决方案来自[https://blog.csdn.net/u014630987/article/details/78670258](https://qinjiangbo.com/how-to-support-mathjax-in-hexo-blogs.html)  
### 2.2 https网站无法显示http的favicon图标  
我的博客是https的，然后加载网站图标的时候报错了
```
The page at 'https://blog.0ggmr0.cn/' was loaded over HTTPS, but requested an insecure favicon 'http://yoursite.com/assets/images/favicon.ico'. This request has been blocked;
```
解决方案来自[https://blog.yangshanlin.cn/2018/08/03/Quick-Start/](https://blog.yangshanlin.cn/2018/08/03/Quick-Start/)的解决方案一  

### 2.3 引入Travis CI
解决方案来自[使用 Travis-CI 持续集成部署 HEXO 博客项目](https://blog.csdn.net/qq_20264891/article/details/82183614)  
`.travis.yml`个人配置如下：  
```yml
os:
  - osx

language: node_js
node_js: stable

cache:
  directories:
    - node_modules

before_install:
  - npm i imagemin-mozjpeg@6.0.0

install:
  - npm install

before_script:
  - git submodule update --remote --merge

script:
  - hexo clean
  - hexo g
after_script:
  - git config user.name "0GGmr0"
  - git config user.email "18817619557@qq.com"
  - sed -i "s/gh_token/${GH_TOKEN}/g" ./_config.yml
  - hexo d
branches:
  only:
    - source-code
env:
 global:
   - GH_REF: github.com/0GGmr0/0GGmr0.github.io

```
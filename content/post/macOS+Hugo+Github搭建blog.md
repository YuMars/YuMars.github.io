---
title: "macOS+Hugo+Github搭建blog"
date: 2023-03-03T15:42:36+08:00
draft: false
--- 

[TOC] 

前几年用`Github`+`Hexo`搭建过Blog，配置的参数和主题过程比较麻烦。折腾好几天，最后发布到`Github`上后过几天莫名其妙打不开主页了，后面没去找原因所以也一直没管。最近看到一个比较方便新的工具`Hugo`，所以重新搭建了一个Blog网站，依然是部署在`Github`上。

# Hugo

> Hugo是由Go语言实现的静态网站生成器。简单、易用、高效、易扩展、快速部署。

[Hugo官网](https://gohugo.io/ "Hugo官网")

[中文官网（翻译不全）](https://www.gohugo.org/ "中文官网")

使用Hugo发布blog简单，不需要很多依赖和额外的配置，很快能将写好的文章发布上去，可以让我把更多的精力花在写blog的内容上。下面来看看具体的步骤。

## 使用Hugo准备：
1. `HomeBrew`安装
>这里不做赘述，毕竟有高墙，会有很多问题，需要一些手段。我遇到的情况是，安装`Hugo`的时候，本来电脑已经有了，但是需要升级，执行了：Running `brew update --preinstall` 需要更新brew好久（这里可以使用国内镜像安装或者更新）。
2. 安装Hugo
```
brew install hugo
```
3. 安装`Git`
3. 安装`Go`：1.18版本以上

## Hugo使用：
### 创建项目：
选择一个文件路径，创建一个新的项目：`quickstart`（quickstart是文件名,可以改成需要的名称）
```
hugo new site quickstart
```
将当前目录更改为项目的根目录。
```
cd quickstart
```
因为需要使用git，所以初始化一个空的git存储库
```
git init
```
clone一个主题（用于网页的最后的样式模板），官方demo使用了`Ananke`主题
```
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke themes/ananke
```
这里也可以去`Hugo`的主题库中挑选一个喜欢的主题样式[https://themes.gohugo.io/](https://themes.gohugo.io/)

在`quickstart`的目录下修改`cofig.toml`文件，更改主题`=。=我是直接打开cofig.toml文件添加的`

```
echo "theme = 'ananke'" >> config.toml
```
启动Hugo，看看效果
```
hugo server
```
### 添加文章内容：
添加一篇新的Blog
```
hugo new posts/my-first-post.md
```
这里面`hugo new`表示创建一篇新的blog，`posts`是对应的目录，`my-first-post`是blog名称，会在`.md`文件自动生成对应标题。一般情况下使用主题，可以直接将themes-主题名-exampleSite-config.toml直接copy到quickstart根目录下。`【这里需要补充的是不同的主题theme，对目录posts有不同的要求，里面主题theme的配置也不同，有的需要根据主题一一对应，有的可以自己生成模块然后做映射。具体可以看看选择的主题里面的配置引导`

刚刚创建的文章`my-first-post.md`在根目录下的`content`中，`content/posts`
```
quickstart
├── archetypes
├── assets
├── config.toml
├── content
├── data
├── public
├── layouts
├── resources
├── static
└── themes
```
打开或者`vim` `my-first-post.md`可以看到
```
---
title: "my-first-post"
date: 2023-03-06T15:42:36+08:00
draft: true
---
```
注意前面`draft`的值是true。默认情况下，`Hugo`不会在构建站点时发布草稿内容，当然这个是可选项，也可以直接把草稿发布到Blog上。

修改、添加完`my-first-post.md`内容后，用命令生成预览的服务器站点（以下命令选一个就行）
```
hugo server                   #
hugo server --buildDrafts     #include content marked as draft
hugo server -D                #include content marked as draft
```

### 网页配置：
打开或`vim`项目根目录中的站点配置文件`config.toml`，根据实际情况更改
```
baseURL = 'https://example.org/' #后面发布的时候需要用到，填yourgithubname.github.io
languageCode = 'en-us'
title = 'My New Hugo Site'
theme = 'ananke'
```
可以选个上面的命令（`hugo server`）继续启动hugo，预览修改效果

### 发布网站：
使用`Github`发布
1. 在`Github`中新建一个仓库，且必须使用`Github`的用户名称，即`yourgithubname.github.io`。
2. 在`setting`中选择`pages`，在`branch`中选择`docs`（用户每次发布的时候自动部署到docs中）
3. 在`config.yaml`或者`config.toml`(这里不同主题用的yaml,toml后缀不同)中加入一行`publishDir = "docs"`。这个影响用户每次发布修改的文章或者新增文章前，自动导出到`docs`目录下，github会自己识别docs目录下的静态页面。
4. 将`quickstart`目录下的所有文件push到`Github`的`yourgithubname.github.io.git`仓库中
```
hugo -D.                 # 重新输出静态网页，默认导出到了public目录下，因为改了publishDir，所以最终去了docs目录下
git add .                 # 全部提交暂存库
git commit -m "input your submit descrpition"        # 将暂存库内容提交本地仓库 加备注
git push                # 同步到远端
```

### 补充：
每次新增文章`hugo new ***/***.md`的时候，根据选择的`themes`不同，在文件目录下需要选择不同的文件，但是`hugo new`之后默认都是新增在了content目录下。最终发布的Blog是在docs目录下的静态网页。当然对前端技术熟悉的话可以轻松定制自己的主题。


```
docs
├── 404.html
├── archives
├── categories
├── css
├── index.html
├── index.xml
├── js
├── page
├── post
├── sitemap.xml
└── tags
```


# 总结：
基本的流程
- 使用hugo的环境配置
- 新建一个hugo项目
- 输入blog内容
- 配置hugo、部署blog文章、预览效果
- 配置github，发布到github（等待1-2分钟后通过yourgithubname.github.io看效果）


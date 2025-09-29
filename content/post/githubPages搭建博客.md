---
title: "github pages 搭建博客"
date: 2022-12-02T09:34:28+08:00
description: ''
tags: [ "github"]
categories: ["tools"]
slug: "set-up-blog-on-github"
draft: false
---

想要找个地方写博客， 最后决定使用github pages搭建。和阮大佬[搭建一个免费的，无限流量的Blog](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)说的一样，经历几个阶段，最后还是想找一个自由不受平台限制的，又不太麻烦的博客平台。

选了一个博客主题，是基于hugo的。

### 步骤

- 安装 [Go](https://go.dev/doc/install)
- 安装hugo `go install -tags extended github.com/gohugoio/hugo@latest`

    报错：go: github.com/gohugoio/hugo@latest: module github.com/gohugoio/hugo: Get "https://proxy.golang.org/github.com/gohugoio/hugo/@v/list": dial tcp 142.251.42.241:443: connectex: A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond.

    需要设置模块代理 `go env -w GOPROXY=https://goproxy.cn`。

    

    再次安装hugo，报错`cgo: C compiler "gcc" not found: exec: "gcc": executable file not found in %PATH%`。需要GCC，在[go huge](https://gohugo.io/installation/windows/#build-from-source)源码安装页面也有说明。

    可以安装mingw，把bin放到环境变量Path中。

    

    输入 `hugo version` 报错。放弃这个方法。直接[下载安装包](https://github.com/gohugoio/hugo/releases/tag/v0.107.0)（intel下载amd即可）。



- github上新建一个 username.github.io 的仓库。（username是自己在github的用户名）。拉取仓库
- 创建站点

    进入仓库的目录，输入 `hugo new site xxx`，新建站点
- 进入xxx/themes, 克隆一个[主题](https://themes.gohugo.io/)。（也可以用子模块引用的方式，不过我受过伤就不用了）

    我用了[这个主题](https://themes.gohugo.io/themes/hugo-theme-diary/)

    `git clone git@github.com:AmazingRise/hugo-theme-diary.git diary`
- 配置，打开xxx/config.toml 增加主题 `theme = 'diary'`
- 本地运行预览，xxx 目录下，`hugo server -D`。访问 [http://localhost:1313/](http://localhost:1313/)。
- 生成使用 `hugo -D` 命令。会将静态网站生成到 public 目录下。`-D` 参数是为了包含草稿文章。
- 新增文章

    `hugo new post/test.md` 在content/post/目录下新增了一个test.md文档。如果自己移动一个md文件进去是不会在网站显示的，因为没有格式说明，如下是yaml的格式：

```text
---
title: "github pages 博客"
date: 2022-12-02T09:34:28+08:00
description: '搭建博客'
tags: [ "github", "blog"]
categories: ["tools"]
slug: "github-pages-blog"
draft: false
---
```





注：如果新建页面，里面文章列表为空的话，页面会显示404。有草稿文章但是发布时没有用`-D`参数也不算。





### 最佳实践：
  1. 将带hugo主题的代码（hugo生成项目和主题拉取代码）放到一个私密库中。
  2. 将生成好的public 代码放到 username.github.io 库中。或者使用子模块（3-4步）
  3. 然后，使用submodule 将后者放到前者的子模块中。` git submodule add  git@github.com:onionc/onionc.github.io.git  ./public`

      （如果有public先删除）

      （重新拉取hugo羡慕时，clone后需要 `git submodule update --init --recursive`来更新子模块）
  4. 在hugo项目下新建一个用来发布博客的脚本

```Bash
#! /bin/bash

echo "---------- 删除(排除.git) -----------"
rm -rf public/* | egrep -v '.git'

#生成
echo "---------- 生成(含草稿) ---------- "
hugo -D

echo "---------- 上传git -------------"
if [ ! -d "public" ]; then
    echo  "public 不存在"
    #exit 0
else
    cd public
    git add .
    git commit -m "update"
    git push origin main
fi

echo "---------  delay 60s ---------- "
sleep 1m
```

1. 上面的3、4步是子模块的方式（用了一段时间感觉不太好用）。如果是两个库分开放，脚本如下：

```Bash
#! /bin/bash

#生成
echo "---------- 生成(含草稿) ---------- "
hugo -D

echo "---------- 删除(排除.git) -----------"
find ../onionc.github.io/* -not -name ".git" | xargs rm -rf

echo "---------- 移动文件夹 ---------- "
cp -rf public/* ../onionc.github.io/

echo "---------- 上传git -------------"
if [ ! -d "../onionc.github.io" ]; then
    echo  "库不存在"
    #exit 0
else
    cd ../onionc.github.io
    git add .
    git commit -m "update"
    git push origin main
fi


echo "---------  delay 60s ---------- "
sleep 1m
```

### 发布文章的流程

1. 将md从某处复制到 `content` 下的某个子目录（也可以是content）。（需要增加格式头。名称和日期必须）
2. 图片放到子目录的`images/` 文件夹，将md中的图片地址改为 `../images`。
3. 点击发布脚本提交博客。
4. 提交hugo项目。



**预览：**

![](../images/20221205/1670134455206_wq1xyXoCeo.jpg)



p.s. 发现一个简洁的主题，很喜欢。[谢益辉](https://yihui.org/)大佬的[xmin](https://github.com/yihui/hugo-xmin)。感谢！

![](../images/20221205/20221205102858.png)

p.p.s. 因为设置了链接格式permalinks，包含有日期，所以之后就算更新了文章也不要轻易更新日期。

### 参考

- [如何利用 GitHub Pages 和 Hugo 轻松搭建个人博客？](https://zhuanlan.zhihu.com/p/57361697)
- [git submodule 目录为空](https://blog.csdn.net/kakadiablo/article/details/119563791)
- 待读[Hugo 不完美教程 - II: Hugo 目录组织](https://www.jianshu.com/p/c5297a8bb1e7)
- [hugo-md文档书写时的格式说明](https://goodmemory.cc/hugo-md%e6%96%87%e6%a1%a3%e4%b9%a6%e5%86%99%e6%97%b6%e7%9a%84%e6%a0%bc%e5%bc%8f%e8%af%b4%e6%98%8e/)
- [Hugo中文文档](https://www.gohugo.org/)





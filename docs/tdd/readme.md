
---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

type: slide
tags: Templates, Talk, slide
slideOptions:
  transition: convex
  theme: Sky
  allottedMinutes: 30
---
<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->
<style>
.reveal section img{border:0}

pre{text-align: left !important;}
.hljs {background: #fefefe;color: #777;}
pre code .gutter.linenumber {
    color: #bfbfbf !important;
    border-right: 3px solid #6DBFFF !important;
}
.reveal pre code {
    max-height: 500px;
}
</style>

# 通过<span style="color: #00BFFF;">测试驱动开发</span>的方式学习go语言

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## 目标

- 通过编写测试学习 Go 语言
- 为测试驱动开发打下基础。
- 可以使用 Go 语言编写健壮的、经过良好测试的系统

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## 学习方法

- 看书
- Coding Kata
- TDD

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

## golang的开发环境

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 下载安装

https://golang.google.cn/dl/

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

#### 环境变量

```sh
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$HOME/bin:$GOPATH/bin:$PATH
```

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 创建workspace

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

```sh
mkdir -p $GOPATH/src $GOPATH/pkg $GOPATH/bin
```

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 安装编译器

```sh
brew cask install visual-studio-code
```

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 安装golang插件

![](/uploads/upload_c591645b8d510a2ac382eee643613729.png)

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 常用插件

```
gocode
godef
golint
go-find-references
go-outline
go-symbols
guru
gorename
goreturns
gopkgs
```

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 插件安装

```sh
go get -u -v github.com/nsf/gocode
go get -u -v github.com/rogpeppe/godef
go get -u -v github.com/golang/lint/golint
go get -u -v github.com/lukehoban/go-find-references
go get -u -v github.com/lukehoban/go-outline
go get -u -v sourcegraph.com/sqs/goreturns
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v github.com/tpng/gopkgs
go get -u -v github.com/newhook/go-symbols
go get -u -v github.com/ramya-rao-a/go-outline
go get -u -v github.com/acroca/go-symbols
go get -u -v github.com/mdempsky/gocode
go get -u -v github.com/rogpeppe/godef
go get -u -v golang.org/x/tools/cmd/godoc
go get -u -v github.com/zmb3/gogetdoc
go get -u -v golang.org/x/lint/golint
go get -u -v github.com/fatih/gomodifytags
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v github.com/sqs/goreturns
go get -u -v golang.org/x/tools/cmd/goimports
go get -u -v github.com/cweill/gotests/gotests
go get -u -v golang.org/x/tools/cmd/guru
go get -u -v github.com/josharian/impl
go get -u -v github.com/haya14busa/goplay/cmd/goplay
go get -u -v github.com/uudashr/gopkgs/cmd/gopkgs
go get -u -v github.com/davidrjenni/reftools/cmd/fillstruct
go get -u -v golang.org/x/net/context/ctxhttp

```

---

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

# 谢谢！



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

# Hello World

<span style="color: #00BFFF;">《通过测试驱动开发的方式学习go语言》--- lnk</span>

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

- 传统的方式
- 如何运行

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

- 如何测试
- 编写测试

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### if

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 变量声明

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### t.Errorf

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### Go 文档

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### Hello, YOU

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 版本控制

每一个基于测试的可用版本都应该提交一次代码

这样你总是可以在重构中陷入混乱的时候回到这个可用版本

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 常量

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 子测试

有时，对一个「事情」进行分组测试，然后再对不同场景进行子测试非常有效。

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 规律


- 编写一个测试
- 让编译通过
- 运行测试，查看失败原因并检查错误消息是很有意义的
- 编写足够的代码以使测试通过
- 重构

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### switch

当你有很多 if 语句检查一个特定的值时，通常使用 switch 语句来代替。如果我们希望稍后添加更多的语言支持，我们可以使用 switch 来重构代码，使代码更易于阅读和扩展。

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### 总结

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### Go 的一些语法

- 编写测试
- 用参数和返回类型声明函数
- if，const，switch
- 声明变量和常量

----

<!-- .slide: data-background="https://i.imgur.com/M2lCepI.jpg" data-background-color="#0E0047" data-background-opacity="0.5"-->

### TDD 过程以及步骤的重要性

- 编写一个失败的测试，并查看失败信息，我们知道现在有一个为需求编写的 相关 的测试，并且看到它产生了 易于理解的失败描述
- 编写最少量的代码使其通过，以获得可以运行的程序
- 然后重构，基于我们测试的安全性，以确保我们拥有易于使用的精心编写的代码

----
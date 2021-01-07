+++
title = "My life in plain text"
tags = ["emacs", "org-mode"]
draft = false
cover = "ox-hugo/org-cover.png"
+++

<!--more-->

这好像是我第三次开始写博客了, 前两次都是水了几篇笔记就放弃了, 一方面是我本来就不
喜欢写东西, 另一方面是感觉写博客太花时间了, 主题、格式、配置一顿折腾, 又写不出什
么有营养的东西😂

但最近沉迷于 emacs 和 org mode, 好像让我找到了做笔记的快感(虽然还没做多少), 所以
打算再尝试一次, 希望能坚持下去

那第一篇文章自然是要推荐一下无敌的 org mode 啦, 事先声明本文主要是为了安利, 展示
一下 org mode 可以干什么, 想看教程的话建议看[官方文档](https://orgmode.org/manual/)


## 什么是 org mode {#什么是-org-mode}

和大家熟悉的 markdown 一样, org mode 也可以通过不同的标记来组织文本的格式、结构,
而且支持的标记和功能更加丰富. 除了写文章, 你还可以用它来管理日程、记录待办事项、
文学编程、写 LaTeX 等等

{{< code language="org" title="官网上的示例" expand="➕" collapse="➖" isCollapsed="true" >}}
#+title:  Example Org File
#+author: TEC
#+date:   2020-10-27

* Revamp orgmode.org website
The /beauty/ of org *must* be shared.
[[https://upload.wikimedia.org/wikipedia/commons/b/bd/Share_Icon.svg]]

** DONE Make screenshots
CLOSED: [2020-09-03 Thu 18:24]

** DONE Restyle Site CSS
Go through [[file:style.scss][stylesheet]]

** TODO Check CSS on main pages [42%]
- [X] Index page
- [X] Quickstart
- [ ] Features
- [ ] Releases
- [X] Install
- [ ] Manual
- [ ] Contribute

* Learn Org
Org makes easy things trivial and complex things practical.

You don't need to learn Org before using Org: read the quickstart
page and you should be good to go.  If you need more, Org will be
here for you as well: dive into the manual and join the community!

** Feedback
#+include: "other/feedback.org*manual" :only-contents t

* Check CSS minification ratios
#+begin_src python
from pathlib import Path
cssRatios = []
for css_min in Path("resources/style").glob("*.min.css"):
    css = css_min.with_suffix('').with_suffix('.css')
    cssRatios.append([css.name,
    "{:.0f}% minified ({:4.1f} KiB)".format( 100 *
                      css_min.stat().st_size / css.stat().st_size,
                      css_min.stat().st_size / 1000)])
return cssRatios
#+end_src

#+RESULTS:
| index.css    | 76% minified ( 1.4 KiB) |
| org-demo.css | 77% minified ( 2.8 KiB) |
| errors.css   | 74% minified ( 4.9 KiB) |
| org.css      | 75% minified (10.7 KiB) |
{{< /code >}}

上面的 org 文件在我的 emacs 上的样子:

{{< figure src="/ox-hugo/2021-01-03_15-01-01_screenshot.png" >}}

除了简单的渲染, org mode 还提供了很多命令和函数, 让你可以对不同类型的文本进行各
种操作


## 基本功能 {#基本功能}

一个 org 文件的基本结构是一棵树, 以此来表现不同段落的层级关系, 你可以很方便地对
不同的节点或子树进行折叠、展开、隐藏等各种操作, 专注于文件的某一部分而不受其他信
息的影响

{{< figure src="/ox-hugo/org-tree.gif" >}}

Org mode 提供了表格编辑功能, 一些基础操作比如: 行列编辑、自动对齐、表格计算等等,
都可以通过简单的命令或者快捷键完成

{{< figure src="/ox-hugo/org-table.gif" >}}

Org mode 的代码块不仅仅提供高亮, 它可以给数十种语言提供统一的执行环境, 传递每个代
码块的执行结果. 简单来说, 你可以用 org mode 进行(文学)编程, 类似于[Jupyter Notebook](https://jupyter.org/)

{{< figure src="/ox-hugo/org-babel.gif" >}}

在 org mode 可以很方便地插入各种类型的链接, 包括网页、文件、邮件、git 仓库等等, 如
果你熟悉 elisp, 甚至可以自定义链接类型, 更改链接的行为. 不过我也是 emacs 和 org mode
的初学者, 对 elisp 还不太熟悉, 感兴趣的话可以看看[官方的示例](https://orgmode.org/manual/Adding-Hyperlink-Types.html)

{{< figure src="/ox-hugo/org-links.gif" >}}

你可以用 org mode 来写几乎任何类型的作品, 因为它提供了一个功能强大、扩展性高的导
出引擎, 你可以自定义导出文件的格式. 同时 org mode 也支持 [Pandoc](https://pandoc.org/) 作为导出工具

因为我很少将 org 文件导出到其他格式中, 所以没怎么折腾这方面的配置, 就只演示一下将
org 文件导出成 html 文档吧

{{< figure src="/ox-hugo/org-export.gif" >}}

Org mode 还有很多功能, 比如你可以给每个标题设置标签、属性、附件、时间戳, 可以很
方便地修改图片大小、导出样式, 还可以用[Drawer](https://orgmode.org/manual/Drawers.html#Drawers)隐藏一些信息, 以及很多我还没了解到的
功能. 下面的章节介绍一些常用的, 且我个人觉得很有帮助的功能和插件, 剩下的就不一一
展示了, 建议有兴趣的直接去看官方手册


## Capture & Refile {#capture-and-refile}

用过 OneNote 的应该知道它有一个"快速笔记", 用来快速记录一些灵感或者摘抄, 待之后
再进行整理. Org mode 的 Capture 同样可以让你在(emacs 上)任何地方进行笔记, 然后自
动保存到你设置好的 org 文件中

Org Capture 更好用的地方是, 你可以自定义"快速笔记"的模板、保存方式, 比如以下设
置:

```emacs-lisp
(setq org-capture-templates
  '(("i" "待办事项" entry (file "~/org/blog/inbox.org")
      "* TODO %?\n %T\n %i\n")
    ("n" "快速笔记" entry (file "~/org/blog/quick-notes.org")
     "* %?\n%x\n" :jump-to-captured t)
    ))
```

这段代码设置了两个 Capture 模板--一个是待办事项, 快捷键是 `i`, 它会自动创建一个 TODO
事项并插入当前的时间戳, 保存到 inbox.org 文件中; 另一个是快速笔记, 快捷键是 `n`, 它
会自动粘贴剪切板的文本到笔记中, 且在完成后跳转到笔记文件

{{< figure src="/ox-hugo/org-capture.gif" >}}

当你需要整理 Capture 笔记时, 可以用 Refile 功能, 它可以将一个段落转移或复制到任意 org
文件中, 或其他段落下. 你也可以用复制粘贴做到这一点, 但 Refile 更优雅😎

{{< figure src="/ox-hugo/org-refile.gif" >}}


## Org Agenda {#org-agenda}

Org Agenda 是一个日程管理工具, 提供了一个界面进行日程相关的操作

前面提到了 TODO 关键字, 实际上你可以自定义任意关键字来说明该事项的状态, 比如:

```emacs-lisp
(setq org-todo-keywords
      '((sequence "TODO(t)" "WAIT(w)" "SOMEDAY(s)" "|" "DONE(d!)" "CANCELED(c@/!)")))
```

以上代码设置了 5 种状态, 后两种意味着该事项已经结束了(完成或取消), `d!` 这个叹号指
当事项切换为 `DONE` 时会插入当前的时间, `c@/!` 不仅插入时间, 还可以写注释说明取消的
原因

除此之外还可以给每个事项设置日期时间、优先级、deadline, 然后通过 Org Agenda 进
行统一管理

{{< figure src="/ox-hugo/org-agenda.gif" >}}

❗ Agenda 的界面是可以自己配置的, 你可以将日程视图改成你喜欢的样子


## 一些扩展包 {#一些扩展包}

Org mode 的另一强大之处, 是它运行在 emacs 这个平台上, emacs 强大的扩展性可以让开发者
为 org mode 创作各种各样的插件, 满足你做笔记、写博客、写论文、管理个人 wiki 甚至写
PPT 等各种需求

下面介绍一些我正在使用的扩展包, 详细的功能请到各自的主页去了解. 除了这些还有很多
扩展包, 有一些我也在用的用来加强原生功能的包我没写出来, 等你们真正使用 org mode 时
自然会去了解了


### org-brain {#org-brain}

Homepage: <http://github.com/Kungsgeten/org-brain>

可以像思维导图一样组织你的笔记, 管理不同笔记间的逻辑关系, 而且提供了一个类似个人
wiki 界面的 visualize 模式

```text
PINNED:  Index


               +-Python              Game development-+-Game design
               +-Programming books           |
   Programming-+-Emacs                       |
         |                                   |
         +-----------------+-----------------+
                           |
                           V
                    Game programming <-> Computer games

Game Maker  Unity

--- Resources ---------------------------------

- https://en.wikipedia.org/wiki/Game_programming
- Passing Through Ghosts in Pac-Man
- In-House Engine Development: Technical Tips

--- Text --------------------------------------

Game programming is the art of programming computer games...
```


### org-journal {#org-journal}

Homepage: <http://github.com/bastibe/org-journal>

写日记专用, 方便自动生成日记, 配置日记模板, 而且提供强大的搜索功能

{{< code language="org" title="示例日记" expand="➕" collapse="➖" isCollapsed="true" >}}
* Tuesday, 06/04/13
** 10:28 Company meeting
Endless discussions about projects. Not much progress

** 11:33 Work on org-journal
For the longest time, I wanted to have a cool diary app on my
computer. However, I simply lacked the right tool for that job. After
many hours of searching, I finally found PersonalDiary on EmacsWiki.
PersonalDiary is a very simple diary system based on the emacs
calendar. It works pretty well, but I don't really like that it only
uses unstructured text.

Thus, I spent the last two hours making that diary use org-mode
and represent every entry as an org-mode headline. Very cool!

** 15:33 Work on org-journal
Now my journal automatically creates the right headlines (adds the
current time stamp if on the current day, does not add a time stamp
for any other day). Additionally, it automatically collapses the
headlines in the org-file to the right level (shows everything if in
view mode, shows only headlines in new-entry-mode). Emacs and elisp
are really cool!

** 16:40 Work on org-journal
I uploaded my journal mode to marmalade and Github! Awesome!

** TODO teach org-journal how to brew coffee
{{< /code >}}


### ox-hugo {#ox-hugo}

Homepage: <https://ox-hugo.scripter.co>

将 org 文件导出为[hugo](https://gohugo.io/)的 markdown 文件, 支持大部分 hugo 配置, 而且你可以选择导出某个子树
为一篇文章, 设置了 TODO 的子树被视为草稿

现在这个博客就是用这个包导出文章的


### org-reveal {#org-reveal}

Homepage: <https://github.com/yjwen/org-reveal>

前面我说可以用 org mode 写 PPT, 并不是真的能写 PPT, 但是 org-reveal 可以借助前端框架
[reveal.js](https://revealjs.com/)将 org 文件导出成一个幻灯片网页

虽然功能上可能没 PPT 那么丰富, 但是绝对可以应付大部分工作展示的需求, 关键是可以
让你从各种字体图片的大小、位置、尺寸问题中解脱, 像写文章一样写幻灯片, 你只需要专
注于内容


### org-drill {#org-drill}

Homepage: <https://gitlab.com/phillord/org-drill>

通过 org mode 实现类似于"记忆卡片"的功能, 可以选择不同记忆算法来制定复习计划, 搭配
org capture 来制作生词本效果很好, 我准备用它来背英语单词

分享一篇教学文章: <https://jmm.io/pr/emacs-meetup/#/>, 这文章就是用 reveal.js 制作的


## 最后 {#最后}

其实自己习惯的工具才是效率最高的, 如果你只是想要一个可以快速上手使用的工具, org
mode 可能不适合你, 因为学习成本太高了(主要是 emacs 上手难)

我个人比较喜欢折腾各种工具软件, 我把 emacs 和 org mode 也是看作需要不断学习的知识,
所以对这种一边用一边学的状态还挺乐在其中的. 但不得不说这很容易让你上头, 耽误你学
习真正需要的知识的时间

即使这篇是安利文章, 最后还是劝你谨慎入坑🙃

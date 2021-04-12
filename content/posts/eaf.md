+++
title = "Emacs Application Framework"
publishDate = 2021-04-02T00:00:00+08:00
tags = ["emacs"]
draft = false
+++

<!--more-->


## What {#what}

EAF 是 [ManateeLazyCat](https://manateelazycat.github.io/) 开发的 Emacs 图形应用框架，通过 PyQt 框架来开发图形应用并将其固定在 Emacs 窗口的合适位置，Emacs 和 Python 进程则通过 IPC 进行通信，达到像操作 Emacs 原生 buffer 和 window 一样操作图形应用的效果。

EAF 使用 QGraphicsScene 和 QGraphicsView 来模拟 Emacs 中的 buffer 和 window，
QGraphicsScene 管理应用的内容、处理键鼠事件，生命周期和 buffer 相同；QGraphicsView
展示图形界面、监听鼠标事件，生命周期和 window 相同，再通过 `QWindow::setParent` 技术将它固定在 Emacs 上。而键盘事件则由 Emacs 接收，通过 EPC 发送给 QGraphicsScene。[^fn:1]

{{< figure src="/ox-hugo/2021-04-02_15-10-55_framework.png" >}}

Homepage: <https://github.com/manateelazycat/emacs-application-framework>


## Why {#why}

不记得之前在哪里看到一句话——Emacs 最大的缺点就是你不得不离开 Emacs。

之前写文章做笔记时，都要一个半屏打开浏览器或者 PDF，一个半屏放 Emacs。现在可以直接在 Emacs 打开浏览器，用来预览本地博客真的很爽。

{{< figure src="/ox-hugo/2021-04-02_16-00-39_screenshot.png" >}}

当然，EAF 的应用比起那些成熟的软件还是有很大差距的，功能也比较简单，不必非要用它来取代其他软件。我个人对 Live in Emacs 是没啥执念的，哪个用着爽就用哪个😁


## How {#how}

EAF 的安装很简单(只要不遇到奇奇怪怪的 bug)，github 主页给了详细的过程，下面我写一下我在 Windows10 & Spacemacs 环境下的安装过程。

① 下载 EAF

```text
git clone --depth=1 -b master https://github.com/manateelazycat/emacs-application-framework.git ~/.emacs.d/private/local/emacs-application-framework/
```

② 安装 python 依赖

```text
cd ~/.emacs.d/private/local/emacs-application-framework/
node install-eaf-win32.js
```

如果安装有问题先检查一下 pip 能不能访问到服务器，连接超时的话一般是代理的锅。

脚本最后会下载一个视频解码器 K-Lite Codec Pack，如果不打算用 Emacs 看视频就可以
`Ctrl-C` 了。

③ 安装 Elisp 依赖

下载安装 emacs-ctable、emacs-deferred、emacs-epc，不过新版 Emacs 好像作为依赖安装好了，反正我是跳过了这一步。

④ 添加配置

```emacs-lisp
(add-to-list 'load-path "~/.emacs.d/private/local/emacs-application-framework/")
(require 'eaf)
```

<details>
<summary>
我的配置，以后能想起来就更新一下。
</summary>
<p class="details">

```emacs-lisp
;; EAF
(use-package eaf
  :load-path "~/.emacs.d/private/local/emacs-application-framework" ; Set to "/usr/share/emacs/site-lisp/eaf" if installed from AUR
  :init
  (use-package epc :defer t :ensure t)
  (use-package ctable :defer t :ensure t)
  (use-package deferred :defer t :ensure t)
  (use-package s :defer t :ensure t)
  :custom
  (eaf-browser-continue-where-left-off t)
  :config
  (setq eaf-fullscreen-p t)
  (eaf-setq eaf-browser-enable-adblocker "true")
  (eaf-setq eaf-browser-dark-mode "false")
  (setq browse-url-browser-function 'eaf-open-browser)
  (defalias 'browse-web #'eaf-open-browser)
  (eaf-bind-key nil "M-q" eaf-browser-keybinding)) ;; unbind, see more in the Wiki

;; eaf-evil
(require 'eaf-evil)
(setq eaf-evil-leader-keymap  spacemacs-cmds)
(define-key key-translation-map (kbd "SPC")
  (lambda (prompt)
    (if (derived-mode-p 'eaf-mode)
        (pcase eaf--buffer-app-name
          ("browser" (if (string= (eaf-call-sync "call_function" eaf--buffer-id "is_focus") "True")
                         (kbd "SPC")
                       (kbd eaf-evil-leader-key)))
          ("pdf-viewer" (kbd eaf-evil-leader-key))
          ("image-viewer" (kbd eaf-evil-leader-key))
          (_  (kbd "SPC")))
      (kbd "SPC"))))

;; eaf-org
(require 'eaf-org)
(defun eaf-org-open-file (file &optional link)
  "An wrapper function on `eaf-open'."
  (eaf-open file))
;; use `emacs-application-framework' to open PDF file: link
(add-to-list 'org-file-apps '("\\.pdf\\'" . eaf-org-open-file))
```
</p>
</details>

[^fn:1]: <https://github.com/manateelazycat/emacs-application-framework/wiki/Hacking>

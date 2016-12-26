---
title: spacemacs使用指南 
date: 2016-12-26 15:59:30
tags:
    - spacemacs
---

# 简介
Spacemacs是一份emacs的配置文件，想要使用它，你先要有emacs，本文主要介绍在Mac上安装和使用Spacemacs，同时本文在Spacemacs基础上使用了子龙山人的Spacemacs-private

# 安装&使用
## Mac平台推荐安装Emacs的方法
{% codeblock lang:shell %}
brew tap railwaycat/emacsmacport;
brew install emacs-mac --with-spacemacs-icon --HEAD --with-gnutls
{% endcodeblock%}
注意：--with-gnutls是用来支持ssl的

## 使用方法

{% codeblock lang:shell %}
#clone spacemacs repo and use develop branch
git clone https://github.com/syl20bnr/spacemacs.git ~/.emacs.d -b develop

#clone zilongshanren layer and checkout develop branch
git clone https://github.com/zilongshanren/spacemacs-private.git ~/.spacemacs.d/
{% endcodeblock %}

详细的配置说明以及安装powershell支持，可以参考[spacemacs-private](https://github.com/zilongshanren/spacemacs-private)

<!-- more -->

# 配置文件
Spacemacs 的配置文件位于 ~/.spacemacs 中，我们只需要修改这个文件就可以制定自己的配置了。
一般情况下，我们只需要在 dotspacemacs-configuration-layers 中添加自己需要的 layer 就可以了。
但是由于我们使用spacemacs-private因此个性化的配置文件都放在了.spacemacs.d目录，可以根据自己的需求修改该目录下的文件

# 常用快捷键
## 配置文件管理
* `SPC f e d` 快速打开配置文件
* `SPC f e R` 同步配置文件
* `SPC q R` 重启 emacs

## 帮助文档
* `SPC h d` 查看 describe 相关的文档
* `SPC h d f` 查看指定函数的帮助文档
* `SPC h d b` 查看指定快捷键绑定了什么命令
* `SPC h d v` 查看指定变量的帮助文档

## 文件管理
* `SPC f f` 打开文件（夹），相当于 `$ open xxx` 或 `$ cd /path/to/project`
* `SPC /` 用合适的搜索工具搜索内容，相当于 `$ grep/ack/ag/pt xxx` 或 `ST / Atom` 中的  `Ctrl + Shift + f`
* `SPC s c` 清除搜索高亮
* `SPC f R` 重命名当前文件
* `SPC b k` 关闭当前 buffer (spacemacs 0.1xx 以前)
* `SPC b d` 关闭当前 buffer (spacemacs 0.1xx 以后)

## 窗口管理
* `SPC f t` 或 `SPC p t` 用 NeoTree 打开/关闭侧边栏，相当于 `ST / Atom` 中的 `Ctrl(cmd) + k + b`
* `SPC f t` 打开当前文件所在的目录
* `SPC p t` 打开当前文件所在的根目录
* `SPC 0` 光标跳转到侧边栏（NeoTree）中
* `SPC n`(数字) 光标跳转到第 n 个 buffer 中
* `SPC w s` 或 `SPC w -` 水平分割窗口
* `SPC w v` 或 `SPC w /` 垂直分割窗口
* `SPC w c` 关闭当前窗口 (spacemacs 0.1xx 以前)
* `SPC w d` 关闭当前窗口 (spacemacs 0.1xx 以后)

## 项目管理
* `SPC p p` 切换项目
* `SPC p D` 在 dired 中打开项目根目录
* `SPC p f` 在项目中搜索文件名，相当于 ST / Atom 中的 Ctrl + p
* `SPC p R` 在项目中替换字符串，根据提示输入「匹配」和「替换」的字符串，然后输入替换的方式：
   + `E` 修改刚才输入的「替换」字符串
   + `RET` 表示不做处理
   + `y` 表示只替换一处
   + `Y` 表示替换全部
   + `n`或`delete`表示跳过当前匹配项，匹配下一项
   + `^` 表示跳过当前匹配项，匹配上一项
   + `,` 表示替换当前项，但不移动光标，可和 `n` 或 `^` 配合使用
   
## 对齐
`SPC j =` 自动对齐，相当于 beautify

## Shell 集成 (必须先配置 Shell layer)
* `SPC '`(单引号) 打开/关闭 Shell
* `C-k` 前一条 shell 命令，相当于在 shell 中按上箭头
* `C-j` 后一条 shell 命令，相当于在 shell 中按下箭头

## 让Spacemacs支持EditorCofnig
EditorConfig 是一个配置文件，一般位于项目的根目录，它可以让不同的编辑器和IDE 都按照相同的格式来格式化代码，对于项目的维护者来说是一个很好的工具。
Spacemacs 也支持 EditorConfig，只需要在配置文件中添加配置即可。下面以 OS X 为例，通过以下步骤即可让 Spacemacs 支持 EditorConfig：

1. `$ brew install editorconfig`
2. 在 `~/.spacemacs.d/init.el` 中的 `dotspacemacs-additional-package`s 中添加 `editorconfig`：
```elisp
dotspacemacs-additional-packages
 '(
   editorconfig
   )
```

3. 创建 `.editorconfig` 文件，写上自己喜欢的配置。
4. 在 `~/.spacemacs.d/init.el` 中的 docspacemacs/user-config` 中加入 `(editorconfig-mode 1)`。


## Git 集成 (必须先配置Magit 的使用)
Git 是一个优秀的版本控制工具，我们可以在 `.spacemacs.d/init.el` 的 `dotspacemacs-configuration-layers` 列表中添加 `git` 就可以集成 `git` 了。

下面是一些常用的 `git` 命令，前缀为 `g`。

spacemacs 0.1xx:

|Git|Magit |
|:---:|:---:|
|`git init`| `SPC g i`|
|`git status` | `SPC g s` |
|`git add` | `SPC g s` 弹出层选中文件然后按`s`             |
|`git add currentFile` | `SPC g S` |
|`git commit` | `SPC g c c` |
|`git push` | `SPC g P` 按提示操作 |
|`git checkout xxx` | `SPC g C` |
|`git checkout -- xxx` | `SPC g s` 弹出层选中文件然后按 `u` |
|`git log` | `SPC g l l` |

spacemacs 0.2xx:

大部分命令整合到 `SPC g s` 中，需要按照提示执行命令

在 commit 时，我们输入完 commit message 之后，需要按 `C-c C-c` 来完成 commit 操作，也可以按  `C-c C-k` 来取消 commit 。


# 插件配置
## 行号开启
默认快捷键是`SPC t n`

如果想开启emacs自动设置, 找到`~/.spacemacs.d/init.el`下的`defun dotspacemacs/config()`在里面添加
```elisp
(global-linum-mode t)
```

## 显示80字符的column
默认快捷键是`SPC t f`

默认启动加入下面
```elisp
 (turn-on-fci-mode)
```
## flycheck
![](http://hackerxu.com/img/spacemacs_flycheck.png)
语法检测, 如上图需要添加`syntax-checking`插件

快捷键`SPC e`, 需要查看error lists使用`SPC e l`

## bookmarks
bookmarks是spacemacs自带的, 可以迅速定位标记的文件, 它可以永久保存
启用的快捷键是`SPC h b`
* 删除书签`C-d`
* 编辑书签`C-e`
* 在另一个窗口打开书签`C-o`

## 搜索
我这里用的ag, 推荐大家使用, 它比ack快一点点

### 搜索内容：
* 和使用vim一样在本文件搜索`/`
* 在项目里智能搜索`SPC /`
* 在所有打开的buffer里搜索`SPC s b`

### 搜索文件名: 
* 在当前目录里搜索文件名`SPC s f`, 其实等价于`SPC f f`
* 最近打开的文件`SPC f r`

## 多光标编辑
需要进入iedit模式, 此时光标变成红色, 步骤如下:
1. 用vim的visul模式选取要replace的值
2. 按`SPC s e`选取全部的匹配值(暂时不知怎么自定义选取)
3. 按`S`对值删除并进行修改
4. 按`ESC ESC`退出

## 注释
* 注释行`SPC c l`
* `SPC c y`这个比较有用, 注释的同时并且复制相同的一份
文档里给出了注释块的快捷方法`SPC ; SPC l`, 其实对于vimer来说使用visul模式选取并用`SPC c l`注释或许是更好的方法.

# 参考资料
[官方文档](https://github.com/syl20bnr/spacemacs/blob/master/doc/DOCUMENTATION.org)
[editorconfig配置](http://brannonlucas.com/using-editorconfig-and-spacemacs-on-os-x/)
[Mode-Hook](https://www.gnu.org/software/emacs/manual/html_node/elisp/Mode-Hooks.html)
[spacemacs-usage](https://scarletsky.github.io/2016/01/22/spacemacs-usage/)
[spacemacs-说明](http://hackerxu.com/2015/08/31/spacemacs.html)

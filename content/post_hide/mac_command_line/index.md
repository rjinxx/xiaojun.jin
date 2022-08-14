---
title: Mac命令行效率提升利器篇
subtitle: ""

# Summary for listings and search engines
summary: 在Mac系统下熟练使用命令行可以使工作更高效，几乎所有的操作都可以用命令行来完成。但这些操作的前提是需要记住各种命令，而且系统原生的终端没有自动补全功能，这让用惯了Xcode的我们尤其不习惯。另外，命令行下路径的切换也显得较为繁琐。工欲善其事必先利其器，本文将介绍一些工具和设置，让命令行使用起来更方便更智能。

# Link this post with a project
projects: []

# Date published
date: "2019-10-08T00:00:00Z"

# Date updated
lastmod: "2019-10-08T00:00:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ''
  focal_point: ""
  placement: 2
  preview_only: false

authors:
- 金小俊

tags:
- Mac OS
- 命令行

categories:
- Mac OS
- 命令行
---

在Mac系统下熟练使用命令行可以使工作更高效，几乎所有的操作都可以用命令行来完成。但这些操作的前提是需要记住各种命令，而且系统原生的终端没有自动补全功能，这让用惯了Xcode的我们尤其不习惯。另外，命令行下路径的切换也显得较为繁琐。工欲善其事必先利其器，本文将介绍一些工具和设置，让命令行使用起来更方便更智能。

### 自动补全

首先我们来给终端命令行加上自动补全的功能，通过[Homebrew](https://brew.sh)安装`bash_completion`即可。当然需要先安装`brew`:

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

在命令行中输入上述命令安装`Homebrew`. 这是一款Mac OS平台下的软件包管理工具，拥有安装、卸载、更新、查看、搜索等众多功能。通过一条指令，就可以实现包管理，而不用关心各种依赖和文件路径的情况。

> Homebrew 会将软件安装到独立目录，并将文件链接至/usr/local路径

安装完`Homebrew`后就可以使用它来安装`bash_completion`了，在终端中输入如下命令:

```
brew install bash-completion
```

安装完成后会提示:

```objc
# Add the following lines to your ~/.bash_profile:
if [ -f $(brew --prefix)/etc/bash_completion ]; then
  . $(brew --prefix)/etc/bash_completion
fi
```

按照提示将上述语句（最后三行）复制到`.bash_profile`文件中。需要注意的是`.bash_profile`为隐藏文件，所以要先显示所有文件，然后在Finder中按快捷键`Command+Shift+G`跳转到该文件。

![image](http://upload-images.jianshu.io/upload_images/268805-7d87efa16df77298?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 默认brew会安装bash-completion，可以先通过`brew list`查看，没有再执行上面的安装步骤。

完成上面的操作之后就可以使用自动补全了，比如我们在某个路径下要打开某个文件，但是忘记那个文件的名字了，或者只记得前几个字母，有了自动补全，我们只需要输入`open`然后直接按`tab`键就会出现目录下的文件了，然后继续按`tab`选择你需要打开的文件名直接回车确认就可以了。效果如下:

![image](http://upload-images.jianshu.io/upload_images/268805-b3b45720f410899b?imageMogr2/auto-orient/strip)

除了系统自有的一些命令补全外，我们还可以把`git`的常用命令也加入到自动补全里面。首先到`git`的[主页](https://github.com/git/git)下载`contrib/completion/`目录下的`git-completion.bash`文件，并将文件放到个人主目录下:

![image](http://upload-images.jianshu.io/upload_images/268805-6c8783b191b478d8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后修改`.bash_profile`文件，在其中添加下列内容:

```
mv git-completion.bash ~/.git_completion.bash
# Add to your .bash_profile:
source ~/.git_completion.bash
```

完成后重新启动下命令行终端就可以使用`git`的自动补全了，效果如下所示:

![image](http://upload-images.jianshu.io/upload_images/268805-38cc9415cd3fc873?imageMogr2/auto-orient/strip)

### 路径切换

在Mac下使用命令行切换路径通常是使用`cd`命令，比如在命令行中输入:

```
cd /Users/Ryan/iOSDev/Documents 
```

即可跳转到`Documents`目录下，但是繁琐的地方在于每次都需要输入完整路径。能不能让命令行记住常用的一些路径且快速切换呢？可以！使用`autojump`就可以实现。`autojump`是一个命令行工具,它允许你直接跳转到你常用的目录,而不受当前所在目录的限制。

`autojump`的安装环境推荐使用`zsh`, `zsh`是`shell`的一种，在Mac OS下默认的`shell`为`bash`, 但其实`zsh`是更强大的`shell`且其完全兼容`bash`, 我们先来看下怎么安装并切换到`zsh`:

```
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

在命令行中输入上述命令即可安装`zsh`, 安装成功后我们需要将系统的默认`shell`设置为`zsh`:

```
chsh -s /bin/zsh
```

这个命令会重启`shell`, 完成后我们在命令行输入:

```
echo $SHELL
```

即可查看当前使用的是哪个`shell` (`bash` or `zsh`).

> shell其实就是一个c语言编写的程序，我们在命令行输入的命令，都是经过shell解释后传送给操作系统（内核）执行。

切换`shell`之后我们可以来安装`autojump`了，还是和上面一样使用`brew`来安装，在命令行中输入如下命令:

```
brew install autojump
```

安装完成后，系统用户根目录下会出现`.zshrc`文件（和上面的`.bash_profile`同一个目录），跳转到这个文件并用文本编辑器打开，在其中找到 `plugins=`, 修改为:

```
plugins=(
    git autojump
)
```

之后新起一行，添加:

```
[[ -s $(brew --prefix)/etc/profile.d/autojump.sh ]] && . $(brew --prefix)/etc/profile.d/autojump.sh
```

修改完内容后`.zshrc`如下图所示:

![image](http://upload-images.jianshu.io/upload_images/268805-68a1577350dc0e22?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

安装完后我们就可以使用`autojump`来快速跳转路径了，在`autojump`中使用`j`(别名)来代替`cd`指令，比如我们想跳转到一个路径，但是我们记不得路径全称，只记得里面有`perfect`这个单词，那么就直接在命令行输入`j perfect`然后按`tab`键，就会出来包含`perfect`的路径，继续按`tab`键选择需要进入的路径再回车确认即可切换到该路径下:

![image](http://upload-images.jianshu.io/upload_images/268805-cc44087b0e497221?imageMogr2/auto-orient/strip)

`autojump`会对访问过的文件和文件夹按照使用频率排序，所以想通过`autojump`快速跳转的路径必须是之前访问过已经被`autojump`记录到数据库中的路径，这样它才会再按照频率列出文件和文件夹。

上文只是对`autojump`基本功能的介绍，至于它的详细功能列表可以到其[主页](https://github.com/wting/autojump)上查看。这边就不再赘述了。另外还一个和功能类似的快速跳转工具`fasd`, 感兴趣的话也可以自行[了解](https://segmentfault.com/a/1190000011327993)下。

除此之外，还有一个赖人软件
`TermHere`, [下载](https://hbang.ws/apps/termhere/)
安装后在任意位置（文件夹上或者文件夹里面目录中）右击鼠标，会发现多了一个菜单项`「New Terminal Here」`点击它就会出现终端窗口，并且当前目录就是你所指的位置。

![image](http://upload-images.jianshu.io/upload_images/268805-6830b3850365cb74?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 命令别名

有时候命令行的命令太长了，我们可以使用一个别名(alias)来代替，类似于程序中的宏。比如我们显示隐藏文件的命令为:

```
defaults write com.apple.finder AppleShowAllFiles true ; killall Finder
```

这个命令太长了，我们可以添加一个别名来代替。打开`.zshrc`文件，在其中添加下列内容:
```
alias sfy="defaults write com.apple.finder AppleShowAllFiles true ; killall Finder"
alias sfn="defaults write com.apple.finder AppleShowAllFiles false ; killall Finder"
```
需要注意的是等号两边均无空格，指令名称中如有空格，需用引号包裹，具体格式为:

```
alias [别名]='[指令名称]'
```

添加完后如下图所示，我们添加了两个别名`sfy`和`sfn`分别表示显示隐藏文件和不显示隐藏文件。在命令行中输入这两个命令和上面的长串命令同等功效。

![image](http://upload-images.jianshu.io/upload_images/268805-4f1a3a9d9d895eb1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

保存并关闭`.zshrc`文件，重新启动命令行后执行别名指令，效果如下所示:

![image](http://upload-images.jianshu.io/upload_images/268805-1713ea553ac2f4f6?imageMogr2/auto-orient/strip)

> 如果没有切换过shell, 还是在系统默认的bash下，则需要把别名的内容添加到bash所对应的.bash_profile文件里面。

### 参考文章
1. [MAC命令行自动补全(git/maven)](https://henulwj.github.io/2016/08/09/mac-cmd-complement/)
2. [Homebrew介绍和使用](https://www.jianshu.com/p/de6f1d2d37bf)
3. [mac安装autojump](https://segmentfault.com/a/1190000011277135)
4. [Mac添加命令别名](https://www.cnblogs.com/shockerli/p/mac-cmd-alias.html)
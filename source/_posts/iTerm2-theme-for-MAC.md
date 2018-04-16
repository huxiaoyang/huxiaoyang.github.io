---
title: iTerm2-Theme-for-MAC
date: 2018-04-16 11:14:43
tags:
  - 工具
---

首先下载安装[iTerm2](https://www.iterm2.com/)

## 1. 安装Oh My Zsh

1、使用 `cat /etc/shells` 命令查看当前系统可以使用哪些shell

``` bash
localhost > cat /etc/shells

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```

2、使用 `echo $SHELL` 命令查看我们当前正在使用的shell

``` bash
localhost > echo $SHELL

/bin/bash
```

3、使用 `chsh -s /bin/zsh` 命令将shell切换为shell之zsh，重启终端生效

4、安装Oh My ZSH

> Tips: 下载前先关闭翻墙代理

``` bash
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

执行完成后，出现以下提示，便是安装成功了

{% asset_img iterm_1.png iterm %}

## 配置主题

[Oh My Zsh提供的所有主题预览](https://github.com/robbyrussell/oh-my-zsh/wiki/Themes)

这里我们选择 `agnster`，修改主题配置：

``` bash
vim ~/.zshrc
直接在配置文件.zshrc中设置主题为"agnoster"
ZSH_THEME="agnoster"
```

### 配置主题需要的依赖

1、[Solarized Dark配色方案](http://ethanschoonover.com/solarized)

下载完成之后解压，在iTerm2的Preferences——Profiles——colors——Load Presets——Solarized Dark即可设置终端配色

{% asset_img iterm_3.png iterm %}

2、[Powerline字体安装](https://github.com/powerline/fonts)

``` bash
// 克隆
git clone git@github.com:powerline/fonts.git
// 安装
./install.sh
```

下载完成之后解压，执行其中的install.sh文件，在iTerm2的Preferences——Profiles——Text中同时将Regular Font和Non—ASCII Font设置为Meslo LG M DZ Regular for Powerline即可

{% asset_img iterm_2.png iterm %}

3、[zsh-syntax-highlighting高亮效果](https://github.com/zsh-users/zsh-syntax-highlighting)

安装：

1. clone仓库：

    ``` bash
    git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
    ```
2. 在 ~/.zshrc中添加插件名称：

    ``` bash
    plugins=( [plugins...] zsh-syntax-highlighting)
    ```
3. 最后，终端执行

    ``` bash
    source ~/.zshrc
    ```

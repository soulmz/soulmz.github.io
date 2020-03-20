---
layout:     post   				    
title:      Homebrew配置 Mirror 阿里云
subtitle:   Homebrew Mirror aliyun
date:       2020-03-17		
author:     soulmz				
header-img: img/backgroup/77642582_p0.jpg	
catalog: true 						
tags:								
    - MacOs 
    - Homebrew
---



> 阿里云的镜像源站：https://developer.aliyun.com/mirror/
>
> 以下内容来源：https://developer.aliyun.com/mirror/homebrew?spm=a2c6h.13651102.0.0.3e221b11XUKvxl

## 简介

Homebrew 是一款自由及开放源代码的软件包管理系统，用以简化 macOS 系统上的软件安装过程。它拥有安装、卸载、更新、查看、搜索等很多实用的功能，通过简单的一条指令，就可以实现包管理，十分方便快捷。

## 配置方法

首先确保你已经安装好了 Homebrew 了, 如果没有, 请参考 OPSX 指引页的 Homebrew 文档；然后你只需要粘贴下述命令在对应终端运行。

### Bash 终端配置

```
    # 替换brew.git:
    cd "$(brew --repo)"
    git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git
    # 替换homebrew-core.git:
    cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
    git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git
    # 应用生效
    brew update
    # 替换homebrew-bottles:
    echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.bash_profile
    source ~/.bash_profile
```

### Zsh 终端配置

```
    # 替换brew.git:
    cd "$(brew --repo)"
    git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git
    # 替换homebrew-core.git:
    cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
    git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git
    # 应用生效
    brew update
    # 替换homebrew-bottles:
    echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.zshrc
    source ~/.zshrc
```

### 恢复默认配置

出于某些场景, 可能需要回退到默认配置, 你可以通过下述方式回退到默认配置。

首先执行下述命令:

```
# 重置brew.git:
	$ cd "$(brew --repo)"
	$ git remote set-url origin https://github.com/Homebrew/brew.git
	# 重置homebrew-core.git:
	$ cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
	$ git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```

然后删掉 HOMEBREW_BOTTLE_DOMAIN 环境变量,将你终端文件

```
 ~/.bash_profile
```

或者

```
 ~/.zshrc
```

中

```
HOMEBREW_BOTTLE_DOMAIN
```

行删掉, 并执行

```
 source ~/.bash_profile
```

或者

```
 source ~/.zshrc
```

## 相关链接

- 下载地址: https://mirrors.aliyun.com/homebrew/
- 官方主页: https://brew.sh/
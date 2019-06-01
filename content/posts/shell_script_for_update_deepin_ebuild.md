---
title: "自动升级deepin ebuild的shell脚本"
date: 2019-06-01T12:14:03+08:00
draft: false
toc: true
images:
tags:
  - Bash gentoo gentoo-dev linux Shell
---

写了一个 [Shell 脚本](https://github.com/zhtengw/ebuildtools/blob/master/version_bump.sh)，用来自动更新 deepin-overlay 中 deepin 软件的ebuild，脚本的主要功能如下：
* 从 deepin-pkg-list 文件读取列表
* 对每个列表中的软件，从 ebuild 中读取当前 overlay 中的最新版本和 HOMEPAGE
* 用 curl 读取 GitHub 仓库中的最新版本
* 比较两个版本，GitHub 仓库有了新版则更新ebuild

脚本中有几个关键的地方需要做个笔记。
##### 读取 ebuild 的版本号
Gentoo ebuild 的文件名组成为 pkgname-<version>[ebuild version].ebuild。这里 pkgname 就是软件名，可能包含数字；<version> 为软件版本，我这里不考虑版本号里带 beta、alpha 之类的情况，就认为版本号是 **数字和句点** 的组合，这是符合 deepin 发布软件的版本情况的；[ebuild version] 是指这个 ebuild 脚本的版本，可能的字符为 “-pre1”“-r1”等等。
脚本中，我读取版本的命令为：
```bash
curVer=$(ls ${catalog}/*.ebuild | xargs -l basename | sed -E 's/.*-([0-9]+[\.0-9+]*).*\.ebuild/\1/' | sort -r -V | head -n1)
```
关键在于 **sed -E 's/.\*-([0-9]+[\.0-9+]\*).\*\.ebuild/\1/'** ， **([0-9]+[\.0-9+]\*)** 匹配版本号格式，即以数字开头，所有组成最多只有数字和句点；为防止匹配到软件名中的数字，在版本号表达式前加上了 "**-**" ；版本号表达式后面加上 "**.\***" 以匹配可能出现的[ebuild version]； **-E** 参数要求 sed 使用扩展的正规表达式，以使用 "\1" 来用版本号取代原字串。

##### 读取 HOMEPAGE
```bash
HOME=$(grep HOMEPAGE ${catalog}/${pkgname}-${curVer}*.ebuild  | sed -E 's/.*"(.*)".*/\1/' | head -n1)
```
因为 ebuild 中 HOMEPAGE 变量的写法为：
```bash
HOMEPAGE="homepage url"
```
所以 **sed -E 's/.\*"(.\*)".\*/\1/' | head -n1)** 的功能是，攫取引号中的内容取代原字串。

##### 读取 Github 仓库中的最新版本
GitHub 官方有 api [get-the-latest-release](https://developer.github.com/v3/repos/releases/#get-the-latest-release)，可以直接读出仓库的最新 release 。但是 deepin 的软件仓库并没有按 GitHub 的格式发布 release ，所以无法使用这个 api 获取最新版本。从一个这个 api 的[应用实例](https://gist.github.com/lukechilds/a83e1d7127b78fef38c2914c4ececc3c#gistcomment-2827298)中得知，GitHub 还有给出 tags 列表的 api ，但实际使用发现，用这个输出的 tags 列表是按照版本号排序的，非发布时间，而 deepin 软件中有很多旧的 tags 是以年份开头的，如果用这个 tags api 的话，这些以年份开头的 tags 会始终置顶。
最后，我使用的方法是，直接下载 https://github.com/linuxdeepin/pkgname/tags 这个页面，从页面源文件中找出 tags 名称：
```bash
repoVer=$(curl --silent ${HOME}/tags | grep -A 1 -E  "a\ href.*/tag/"  | grep -v "/tag/" | sed s/[[:space:]]//g | head -n1)
```

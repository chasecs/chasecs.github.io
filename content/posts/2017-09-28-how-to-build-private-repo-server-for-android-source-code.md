---
layout: post
categories: ["git"]
date: 2017-09-28
title:  "搭建支持 Repo 的 Android 源码镜像（Repo 服务器）"
description: "方案厂商给了一份 Android 源码，没有 manifest.git 文件，不支持 Repo。为了基于这份代码搭建支持 Repo 的镜像服务器，断断续续摸索了两个星期，总算 hacking 成功。"
---

<!-- # 搭建管理 Android 源码的 Repo 服务器 -->

方案厂商给了一份 Android 源码，没有 manifest.git 文件，不支持 Repo。为了基于这份代码搭建支持 Repo 的镜像服务器，断断续续摸索了两个星期，总算 hacking 成功。

本文用到的主要知识：
- shell script
- git 指令

## 一、关于 Repo

基于 Android 源码的开发工作大多要用到 Git 和 Repo。

[Repo](https://source.android.com/source/developing) 是基于 Git 的仓库管理工具，支持同时管理许多个 Git 仓库。因为 Android 源码包含了许多个 Git 仓库，使用 Repo 可以简化许多工作。比如，使用一个 Repo 命令，就可以从多个不同的仓库下载文件，同步到你的计算机上。

搭建支持 Repo 的 Android 源码镜像，主要步骤如下：
1. 在服务器搭建 Git 托管服务器
2. 在客户端安装配置好 Repo
3. 在客户端创建 manifest/default.xml 并上传到 Git 服务器
4. 将客户端 Android 源码上传到 Git 服务器
5. 在其它获得 git 权限的客户端使用：Repo init; Repo sync

## 二、搭建 Git 服务器

搭建 Git 服务器这部分的内容相对独立，和 Repo 的关系不大，因此另外写了一篇文章：

[在 ubuntu 搭建基于 Gitolite 的 Git 服务器](https://chasecs.github.io/2017/09/26/gitolite-installation-and-setup-on-ubuntu-16-04-lts.html)

## 三、搭建 Repo 服务器的设备及要求

服务器 A：
- IP 地址：192.168.1.101
- 安装了 Gitolite   
- 将作为 Repo 服务器（这样称呼比较直观，但不知是否准备）

客户端 B：
- IP 地址：192.168.1.102
- 取得 A 的 Gitolite 的管理员身份（可以修改 gitolite-admin)
- 存放了厂商给的 Android 源码 `asop/`，没有 `manifest/default.xml` 文件

以下的操作，除非有特别说明，都在客户端 B 执行。

## 四、安装 Repo

Repo 安装可以参考[Installing Repo](https://source.android.com/source/downloading)。主要步骤如下：
```bash
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

因为特殊的国情。。。上述操作后 Repo 还是很难直接使用的。比如在 `repo init` 时，即使是科学上网，也无法连接上
```
REPO_URL = 'https://gerrit.googlesource.com/git-repo'
```

一个替代的方案是，使用清华的镜像。打开 `~/bin/repo`，把里面的 `REPO_URL` 地址改为：
```
REPO_URL = https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/
```

这样就可以运行 `repo` 指令了。

## 五、认识 manifest/default.xml 文件

Android 源码里有上百个 git 项目，不同版本的源码项目各不相同，Repo 指令如何知道具体有哪些项目呢，答案就在  `manifest/default.xml ` 。 `default.xml` 记录了这些 git 项目的名称、路径等信息，通过它， Repo 就有迹可循。

了解 `manifest/default.xml `最好的办法，就是从零搭建一个 Repo 服务器。

随便找台 Linux 操作系统的计算机，新建一个目录模拟 Repo 服务器：
```bash
mkdir /tmp/repo-server
cd /tmp/repo-server
```

在 `/tmp/repo-server` 目录创建若干个 git 仓库:
```bash
#创建 git 仓库 project_1
mkdir project_1 && cd project_1
git init;
touch p1.txt;
git add .;
git commit –m "initial commit"

cd..

#创建一个 git 仓库 project_2
mkdir project_2 && cd project_2
git init;
touch p2.txt;
git add .;
git commit –m "initial commit"
```

在 `/tmp/repo-server` 创建 manifest 仓库：
```
#创建 manifest 仓库
mkdir manifest && cd manifest
git init;
touch default.xml;
```

在 `default.xml` 加入以下内容：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote  name="test"
           fetch="." />
  <default revision="master"
           remote="test"
           sync-j="4" />
  <project path="project_1" name="project_1" />
  <project path="project_2" name="project_2" />
</manifest>

```

然后 commit 代码
```bash
git add .;
git commit –m "initial manifest"
```

这样，模拟 Repo 服务器就搭建完成了。

接着再建一个新的目录`/tmp/repo-client/`，模拟 Repo 客户端。在客户端目录里：
```bash
repo init –u /tmp/repo-server/manifest
```

如果一切正常，将会收到成功提示信息。这时候去查看该目录下的 .repo，就会发现有一个 `manifest.xml`，内容是克隆服务器的 `default.xml` 文件。

最后，使用 Repo 指令将多个 git 项目的代码一次性同步到客户端上：
```bash
repo sync
```

仔细揣摩 default.xml 文件的内容，你应该能理解这个 xml 文件的作用。

*注意：以上试验，我在 centOS 测试成功，在 macOS 测试时，执行 `repo init -u $LOCAL_ADDRESS` 会报错，会要求最后一个参数是 url*

## 六、从源码 `aosp/` 找出所有 git 仓库

从厂商得到的 Android 源码 `aosp/` 目录大致如下，
```bash
aosp
|- art
|- /abi
  |- cpp
|- /developers
  |- build
  |- demos
  |- samples
    |- /android
...
```

因为厂商不用 Repo 管理，所以源码里也没有 `manifest/default.xml` 文件。

幸好，仔细查看 `aosp/` 之后发现，里面有些目录下有 `.git/` 目录，说明它就是一个 git 项目。因此可以通过找出所有带 `.git/` 目录的目录，来确定 `aosp/` 有哪些 git 项目。代码如下：
```bash
find aosp/ -type d -name '.git' > git_projects.txt`
```

最终在这份 `aosp/` 共找到 531 个 git 项目。


## 七、创建 default.xml 文件

得到 `git_projects.txt` 后，就可以据此创建 `default.xml` 文件。

`git_projects.txt` 每一行的内容是这样的：
```bash
aosp/prebuilts/devtools/.git
```

使用 bash 指令去掉最后开头的 `aosp/` 和末尾的 `/.git` ：
```
cat git_projects.txt | cut -c 6- | sed 's/.....$//' > path.txt
```

得到每一行内容的格式如下：
```
prebuilts/devtools
```

生成 xml 文件的脚本 gen_xml.sh ：
```
#!/bin/bash

echo -e "
<?xml hhhversion=\"1.0\" encoding=\"UTF-8\"?>
<manifest>
  <remote  name=\"aosp\"
           fetch=\".\"/>
  <default revision=\"master\"
           remote=\"aosp\"
           sync-j=\"4\" />" >>$1

while read line; do
	echo "<project path=\"$line\" name=\"$line\" />" >>$1
done
echo -e "\n</manifest>" >>$1
```

bash 指令创建 `default.xml`：
```
cat path.txt | ./gen_xml.sh default.xml
```

得到的 `default.xml` 里，`<project />` 示例如下：
```
<project path="prebuilts/devtools" name="prebuilts/devtools" />
```

## 八、在 Gitolite 初始化所有 `asop/` 的 git 仓库

**前提：客户端 B 可以修改服务器 A 的 gitolite-admin 项目，即管理 Gitolite 的项目**

这里还要用到 `git_projects.txt` 文件。这次只要去掉每行末尾的 `/.git`：
```
cat git_projects.txt | sed 's/.....$//'  > repo_path.txt
```

得到的每一行的格式如下：
```
aosp/prebuilts/devtools
```

编写 gen_server_repo.sh
```bash
#!/bin/bash

# 声明群组 @aosp_dev , 成员: jack, tom
@aosp_dev    =    jack tom

# 加上 manifest 仓库
echo -e "
repo aosp/manifest\n
     RW+    =    @aosp_dev\n" >>$1

while read line; do
	echo -e "repo $line \n     RW+    =    @aosp_dev\n" >>$1
done

```

bash 指令生成 `aosp.conf` 文件:
```bash
cat repo_path | ./gen_server_repo.sh  aosp.conf
```

在 `gitolite-admin/conf/gitolite.conf` 开头加入：
```bash
include "aosp.conf"
```

把更新的内容 push 到服务器 A ，gitolite 就会在相应目录（通常是 `/home/git/repositories` ）初始化所有 `aosp` 的所有 git 仓库。

至此，aosp 的所有仓库已经在服务器 A 生成了，下一步就是把 aosp 的源码上传到服务器。

## 九、上传 `manifest/default.xml` 到服务器

把之前创建好 `default.xml` 文件上传到服务器的 `aosp/manifest` 仓库：
```bash
git clone git@192.168.1.101:aosp/manifest
cd manifest
#
# 把default.xml 文件放到 manifest/ 目录
#
git add .
git commit -m 'add default.xml'
git push
```

## 十、上传 `aosp/` 源码到服务器

上传源码的 shell 脚本 `push_aosp.sh`:
```shell
#!/bin/bash

work_dir=$1

pwd=${PWD}
count=0

while read line; do
    echo $line
    count=$((count+1))
    line1=${line%%/*}
    if [ -z "$line" ]; then
        echo $work_dir not exist !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! 1>&2
        continue
    fi
    if [ $(ls -A $pwd/$line | wc -l) -eq 0 ]; then
        echo $work_dir empty !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!! 1>&2
        continue
    fi
    workdir=$pwd/$line
    cd $workdir
        rm -rf .git
        git init .  1>&2
        git add . -f 1>&2
        { git commit -m "Initial commit" &&
            git push -f --set-upstream git@192.168.1.101:$line.git master
        } || {
            touch empty_file
            git add .
            git commit -m "添加一个空文件，消灭空仓库"
            git push -f --set-upstream git@192.168.1.101:$line.git master
            echo number:$count should be empty $line >> $HOME/log_$(date +%Y_%m_%d)
        }
	echo -e "number:$count \n"
    cd -
done
```

注意 `push_aosp.sh` 里的这段脚本：
```shell
{ git commit -m "Initial commit" &&
    git push -f --set-upstream git@192.168.1.101:$line.git master
} || {
    touch empty_file
    git add .
    git commit -m "添加一个空文件，消灭空仓库"
    git push -f --set-upstream git@192.168.1.101:$line.git master
    echo number:$count should be empty $line >> $HOME/log_$(date +%Y_%m_%d)
}
```

这其实是个变通办法。因为之前试过忽略空仓库，不做 push 。但代码上传到服务器后，客户端使用 `repo sync` 下载时，会出现 `error: Exited sync due to fetch errors` 错误，导致同步失败。

所以只好消灭所有空仓库。

这里还把所有加工过的空仓库记录到 `$HOME/log_$(date +%Y_%m_%d)` 文件里，作为备忘。

最后，把 `repo_path.txt`，`push_aosp.sh` 放在和源码 `aosp/ ` 同一个目录里，然后执行：
```
cat repo_path.txt | ./push_aosp.sh
```

在我的例子里，源码大小有 22G ，上传花了 2 个多小时。

## 十一、其它客户端使用 Repo 服务器

源码成功上传到服务器 A 之后，其它客户端就可以下载使用了。假设客户端计算机 C 要使用，流程如下：

1. C 先获得服务器 A 上的相关 repo 的权限，具体参考[在 ubuntu 搭建基于 Gitolite 的 Git 服务器](https://chasecs.github.io/2017/09/26/gitolite-installation-and-setup-on-ubuntu-16-04-lts.html)
2. 安装 Repo ，具体参考上文第二节“安装 Repo”
3. `repo init -u git@192.168.1.101:aosp/manifest`
4. `repo sync`


## 参考资料

- [What is Repo and Why does Google use it?](https://stackoverflow.com/questions/2450601/what-is-repo-and-why-does-google-use-it)
- [How does the Android repo manifest repository work?](https://stackoverflow.com/questions/6149725/how-does-the-android-repo-manifest-repository-work)
- [Creating a private repo server for Android](https://groups.google.com/forum/#!topic/repo-discuss/qXBILfjVV04)
- [Repo command reference](https://source.android.com/source/using-repo)
- [Downloading the Source](https://source.android.com/source/downloading)
- [default.xml](https://android.googlesource.com/mirror/manifest/+/master/default.xml)
- [Create your own Repo Server](http://www.programering.com/a/MTMzADNwATc.html)
- [Build local repo server](http://www.phonesdevelopers.info/1707216/)
- [清华大学开源软件镜像站](https://mirror.tuna.tsinghua.edu.cn/help/AOSP/)
- [Gitolite + repo 搭建安卓源码开发环境](http://blog.csdn.net/u011479494/article/details/50629669)
- [Android Repo的manifest XML文件格式](http://blog.csdn.net/hansel/article/details/9798189)

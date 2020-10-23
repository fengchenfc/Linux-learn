# 版本控制

## 概念

#### 版本库

客户/服务器系统

- 版本库时版本控制的核心
- 任意数量客户端
- 客户端通过写数据库分享代码

![image-20201022184503397](C:\Users\Administrator\Desktop\Linux-note\image-20201022184503397.png)

#### 分布式版本控制

##### 集中式版本控制系统

- 开发者之间公用一个仓库（repository）
- 所有操作都需要联网

> svn	集中式	用户使用该服务时，需要时刻与服务器保持在线状态，数据统一保存在svn服务器中

##### 分布式版本控制系统

- 每个开发者都是一个仓库的完整克隆，每个人都是服务器
- 支持断网操作

> git	分布式	用户使用该服务时，不需要时刻与服务器保持在线状态，仅仅传递数据时需要联网，数据保存在git服务器与git客户端

-------

## git

### git概念

- git仓库：保存所有数据的地方
- 工作区：从仓库中提取出来的文件，放在磁盘上供使用/修改
- 暂存区：一个文件，索引文件，保存了下次将提交的文件列表信息

![image-20201022185019935](C:\Users\Administrator\Desktop\Linux-note\image-20201022185019935.png)



> https://gitee.com/
> https://github.com/

## git服务器端

### 1.安装git软件

```shell
yum -y install git
```

### 2.创建服务器版本仓库

服务器是一台多人协作的中国内心服务器

init初始化一个空仓库（没有具体数据）

#### git init 与git init --bare

- git init

git init 创建工作仓库（working repository）

适合于实际编辑生产过程中，在工作目录下，你将可以进行实际的编码、文件管理操作和保存项目在本地工作。**也就是说，在远程仓库上代码，是否需要编辑和执行，如果需要执行，就使用git init**

- git init --bare

git init --bare创建裸仓库（bare repository）

主要是用作分享版本库,这个仓库只保存git历史提交的版本信息，而**不允许用户在上面进行各种git操作**，如果我们进行其他操作的话，会得到诸如

This operation must be run in a work tree。

```shell
mkdir -p /var/lib/git
git init /var/lib/git/project --bare  //创建空仓库Project
ls /var/lib/git/project
```



##  git客户机端

### 1.安装git软件

同服务端安装

### 2.使用git

```shell
[root@web2 ~]#git clone 192.168.2.100:/var/lib/git/project  //在客户机克隆服务器的仓库
[root@web2 ~]#cd  project  //进入仓库
[root@web2 project]# echo "web2_01" > web2_01.txt   //创建测试文件   
[root@web2 project]# git add .   //提交到暂存区
[root@web2 project]# git commit -m "web2_01.txt"  //将文件保存到仓库中，并添加日志提示信息
```

首次保存会失败，因git需要下述客户端用户标记信息，输入下列两个配置命令

```shell
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

再次提交文件保存到仓库中

```shell
[root@web2 project]# git commit -m "web2_01.txt"  //再次提交文件保存到仓库中
[root@web2 project]#git push     //将本地仓库中的数据推送到远程服务器
[root@web2 project]# git config --global push.default simple  //配置使用习惯
[root@web2 project]#git push      //push文件到远程服务器
```

### 3.访问方式

本地访问

- git clone file:///var/lib/git/

远程ssh访问

- git clone root@服务器IP:/var/lib/git

web访问

- 服务器需要额外配置web服务器
- 客户端可以浏览器访问
- git clone http://服务器IP/git仓库
- git clone https://服务器IP/git仓库

### 4.客户端支持的子命令操作

- clone		//将远程服务器的仓库克隆到本地
- config        //修改git配置
- add             //添加修改到暂存区
- commit         //提交修改到本地仓库
- push              //提交修改到远程服务器

### 5.git日志分析

```shell
git log   //查看完整日志
git log --pretty=oneline  //查看精简日志
git log --oneline  //查看最精简日志
git reflog  //查看本机操作记录
```

## HEAD指针

head指针时一个可以再任何分支和版本移动的指针

通过移动指针，我们可以将数据还原至任何版本

### 移动head指针

先使用log指令查看版本信息

```shell
[root@web2 project]# git log --oneline  //查看日志
```

回到目的版本,xxxx是根据日志显示所得到的目的版本的节点信息

```shell
[root@web2 project]# git  reset  xxxx  --hard  //回到过去的某个记录，其xxxx”是日志中显示的时间节点信息，要根据实际修改
```

查看恢复记录之后的日志记录

```shell
[root@web2 project]# ls

[root@web2 project]# git reflog  //查看回复记录之后的日志记录
```

###### 恢复到过去的时间节点，找回数据

思路：

1. git log --oneline  查看日志，找到旧数据所在时间节点
2. git  reset  xxxx  --hard  回到过去，xxxx是时间节点的记录
3. 把需要找回的数据，从仓库中拷贝到另外一个目录
4. git  reset  xxxx  --hard  回到现在
5. 在之前的目录找回旧数据

## git分支

当项目内容比较多时，可以在git中使用分支，不同分支的文件可以互不干扰而不用创建多个仓库

### 基本概念

分支可以让开发分多条主线同时进行，每条主线互不影响

- 按功能模块分支、按版本分支
- 分支也可以合并

![image-20201022192512690](C:\Users\Administrator\Desktop\Linux-note\image-20201022192512690.png)

- 常见的分支规范
  - MASTER分支（MASTER是主分支，是代码的核心）
  - DEVELOP分支（DEVELOP最新开发成果的分支）
  - RELEASE分支（为发布新产品设置的分支）
  - HOTFIX分支（为了修复软件BUG缺陷的分支）
  - FEATURE分支（为开发新功能设置的分支）

### 使用方法

查看当前分支，*为所在位置

```shell
[root@web2 project]# git branch  查看当前分支，*是所在位置
```

创建hotfix分支

```shell
[root@web2 project]# git branch hotfix 创建hotfix分支
```

切换到hotfix分支

```shell
root@web2 project]# git checkout hotfix 切换到hotfix分支
```

合并分支，将hotfix合并到master分支

**合并前，一定要切换到master分支！**

执行merge命令合并分支

```shell
root@web2 project]# git checkout master 切换到hotfix分支
root@web2 project]# git merge hotfix  合并master与hotfix
```

### 解决不同分支文件冲突

如果分别在不同分支，创建同名文件，内容不同，再进行合并时，会发生冲突，此时需要手工修改冲突文件，修改完之后，就可以解决冲突

```shell
git checkout hotfix   //先切换到hotfix分支
echo abc > abc.txt   //创建文件
git add .   //提交到暂存区
git commit -m "abc.txt"   //提交到仓库保存
git checkout master  //再切换到主分支
echo xyz > abc.txt   //创建同名文件，但内容不同
git add .    //提交到暂存区
git commit -m "abc.txt"    //提交到仓库保存
git merge hotfix    //合并时会发生冲突

[root@web2 project]# cat a.txt     //该文件中包含有冲突的内容
<<<<<<< HEAD
BBB
=======
AAA
>>>>>>> hotfix
[root@web2 project]# vim a.txt  //修改该文件，为最终需要的数据，解决冲突
BBB

[root@web2 project]# git add .
[root@web2 project]# git commit -m "resolved"
```



## git服务器协议

### SSH协议

git默认使用ssh协议工作，使用密码认证访问

客户端使用ssh远程访问（可读写权限）

#### 免密远程git服务器（密钥授权）

配置ssh密钥，实现无密码访问git服务器

```shell
ssh-keygen -f /root/.ssh/id_rsa -N ''  //在客户端创建秘钥
秘钥没有密码
ssh-copy-id 192.168.2.100   //传入服务器
git clone 192.168.2.100:/var/lib/git/abc  //克隆git服务器仓库，已经无需密码
cd abc   //进入仓库
echo abc > abc.txt     //创建测试文件 
git add .  //提交到暂存区
git commit -m "abc"   //提交到仓库保存
git push   //推送到git服务器，也不需要密码了
```

### git协议

git协议访问支持无授权访问（只读）

- 服务器安装git-daemon软件包

```shell

```

- 服务器初始化仓库（必须在/var/lib/git/目录创建仓库）

```shell

```

- 服务器启动git服务

```shell

```

- 客户端使用git协议访问（只读）

```shell

```



### http协议









--------


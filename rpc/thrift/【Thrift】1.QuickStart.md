# 【Thrift】1.QuickStart

公司的一些平台服务框架底层封装了thrift提供服务,最近项目不是很紧,于是研究了一下,刚刚入门,理解得不深,写这篇博文来整理一下思路.

## 一、什么是thrift?

　　简单来说,是Facebook公布的一款开源跨语言的RPC框架.

　　那么问题来了.

### 1.1 什么是RPC框架?

RPC全称为Remote Procedure Call,意为远程过程调用.

假设有两台服务器A,B。A服务器上部署着一个应用a, B服务器上部署着一个应用b, 现在a希望能够调用b应用的某个函数(方法), 但是二者不在同一个进程内, 不能直接调用, 就需要通过网络传输, 在AB服务器之间建一条网络传输通道, a把参数传过去, b接收到参数调用自己的方法, 得到结果, 再通过网络传回给a, 简单讲就是 A 通过网络来调用 B 的过程. 这个过程要涉及的东西很多, 比如多线程, Socket, 序列化反序列化, 网络I/O, 很复杂, 于是牛掰的程序员把这些封装起来做成一套框架, 供大家使用, 就是RPC框架.　　　　　　　

### 1.2 thrift的跨语言特型

thrift 通过一个中间语言 IDL(接口定义语言)来定义RPC的数据类型和接口, 这些内容写在以 .thrift 结尾的文件中, 然后通过特殊的编译器来生成不同语言的代码, 以满足不同需要的开发者, 比如java开发者,就可以生成java代码, c++ 开发者可以生成 c++ 代码,生成的代码中不但包含目标语言的接口定义, 方法, 数据类型, 还包含有 RPC 协议层和传输层的实现代码.

### 四、thrift的协议栈结构

![](https://images2015.cnblogs.com/blog/870109/201702/870109-20170221155000163-876398090.png)

thrift 是一种 c/s 的架构体系. 在最上层是用户自行实现的业务逻辑代码.第二层是由 thrift 编译器自动生成的代码，主要用于结构化数据的解析，发送和接收。**TServer** 主要任务是高效的接受客户端请求，并将请求转发给**Processor** 处理。**Processor** 负责对客户端的请求做出响应，包括RPC请求转发，调用参数解析和用户逻辑调用，返回值写回等处理。从 **TProtocol** 以下部分是 thirft 的传输协议和底层 I/O通信。**TProtocol** 是用于数据类型解析的，将结构化数据转化为字节流给 **TTransport** 进行传输。**TTransport** 是与底层数据传输密切相关的传输层，负责以字节流方式接收和发送消息体，不关注是什么数据类型。底层IO负责实际的数据传输，包括socket、文件和压缩数据流等。

## 二、MAC OS下thrift的下载与安装

我的电脑是mac, 第一次安装也碰到了一些问题, 所以有必要记录一下.

首先,在官网下载安装包 http://thrift.apache.org/download 截止2021-06-22. 官网是 0.14.2 版本.下载完之后解压到想要安装的目录.

进入根目录:

```sh
# sherlock @ mbp in ~/devTools/opt [11:14:15]
$ cd thrift-0.14.2

# sherlock @ mbp in ~/devTools/opt/thrift-0.14.2 [11:15:22]
$ ./configure
......
checking for bison... yes
checking for bison version >= 2.5... no
configure: error: Bison version 2.5 or higher must be installed on the system!
```

安装的时候,第二步出现了问题,提示:

> **Bison version 2.5 or higher must be installed on the system!**

原因是 Bison 版本过低,mac 默认安装的版本是 2.3,因此需要安装最新版的 Bison,命令行输入:**brew install bison** 安装最新版 bison,如果你的命令行反馈:**Command not found**,很可能是因为你没装homebrew, 命令行输入:`ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`

等待安装完 homebrew 就可以安装 bison 了,最新版本 3.7.5 安装完以后

```sh
$ brew install bison
Updating Homebrew...
Warning: bison 3.7.5 is already installed and up-to-date.
To reinstall 3.7.5, run:
  brew reinstall bison
```

执行第二步, 发现依然提示上面那个警告,原因是因为它读取的仍然是默认的 bison,于是找到系统安装的 bison 目录,我的 mac 是 `/Library/Developer/CommandLineTools/usr/bin/bison`, 解决方法也比较简单, 可以先将这个目录下的 bison 名字改一下, 再将最新版的 bison 复制进来, 于是, 在bin目录下, 执行命令:

```shell
# sherlock @ mbp in /Library/Developer/CommandLineTools/usr/bin [11:21:04]
$ sudo mv bison bison_bak

# sherlock @ mbp in /Library/Developer/CommandLineTools/usr/bin [11:22:11]
$ sudo cp /usr/local/opt/bison/bin/bison /Library/Developer/CommandLineTools/usr/bin/ 
```

现在, 再按照上面的步骤进行下去, 就可以发现还有好多坑:

1. Xcode 需要正确安装(这个百度就 OK)

2. 本地最好不要有 Ruby, 不然大概率会有版本冲突

3. 如果出现以下错误: 对不起, 我也没解决这个错误......

   ```sh
   # sherlock @ mbp in ~/devTools/opt/thrift-0.14.2 [14:56:01]
   $ make
   /Applications/Xcode.app/Contents/Developer/usr/bin/make  all-recursive
   Making all in compiler/cpp
   Making all in src
   /bin/sh ../../../ylwrap thrift/thrifty.yy y.tab.c thrift/thrifty.cc y.tab.h `echo thrift/thrifty.cc | sed -e s/cc$/hh/ -e s/cpp$/hpp/ -e s/cxx$/hxx/ -e s/c++$/h++/ -e s/c$/h/` y.output thrift/thrifty.output -- bison -y -d
   /Users/sherlock/devTools/opt/thrift-0.14.2/compiler/cpp/src/thrift/thrifty.yy:1.1-5: invalid directive: `%code'
   /Users/sherlock/devTools/opt/thrift-0.14.2/compiler/cpp/src/thrift/thrifty.yy:1.7-14: syntax error, unexpected identifier
   make[3]: *** [thrift/thrifty.cc] Error 1
   make[2]: *** [all-recursive] Error 1
   make[1]: *** [all-recursive] Error 1
   make: *** [all] Error 2
   ```

最后的最后, 使用了 homebrew 安装 thrift, 因为很久没用 homebrew, 升级了一下 homebrew, `brew update`, 并且将中科大的镜像源切换为清华的镜像源

```sh
# 替换brew.git源
git -C "$(brew --repo)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git
# 替换 homebrew-core.git源
git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
# 替换 homebrew-cask.git源
git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask.git
# 更新
brew update

# 修改 ~/.bash_profile
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile
```

安装 thrift:

```sh
# sherlock @ mbp in ~ [16:06:47]
$ brew install thrift
Updating Homebrew...
==> Downloading https://mirrors.aliyun.com/homebrew/homebrew-bottles/thrift-0.14.1.big_sur.bottle.tar.gz

curl: (22) The requested URL returned error: 404
Warning: Bottle missing, falling back to the default domain...
==> Downloading https://ghcr.io/v2/homebrew/core/thrift/manifests/0.14.1
Already downloaded: /Users/sherlock/Library/Caches/Homebrew/downloads/7978b68049c5b2f2666fa5b87f159802f9cfc6e3828818abdb054f938629676f--thrift-0.14.1.bottle_manifest.json
==> Downloading https://ghcr.io/v2/homebrew/core/thrift/blobs/sha256:b408f5d8714788cb4d5ffa575718b78fa63233663b33d6932e3611e577a087eb
Already downloaded: /Users/sherlock/Library/Caches/Homebrew/downloads/a1760de39ca546c24e76d7576ca9f57a2b2783e7feefc403f98498fd042ccd49--thrift--0.14.1.big_sur.bottle.tar.gz
==> Pouring thrift--0.14.1.big_sur.bottle.tar.gz
🍺  /usr/local/Cellar/thrift/0.14.1: 103 files, 7MB
```

这是这么 easy......

```sh
# sherlock @ mbp in /usr/local/Cellar [16:08:34] C:1
$ thrift -version
Thrift version 0.14.1
```




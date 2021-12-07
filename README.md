# ALICE环境搭建教程

ChunzhengLab

请同时参考[ALICE Analysis Tutorial](https://alice-doc.github.io/alice-analysis-tutorial/)。该文档中一切与本文不一致的内容，以该文档为准

## 1. Git与Github

###  1.1 基概基操

1. git概念、版本控制系统、github

2. 常见命令：git init git status、git add 、git commit、git branch、git checkout 、git merge

3. 远程仓库：git pull、 git push、git clone(http、ssh) — http链接每次push需要输入帐号密码，ssh上传客户端公钥可以直接push与pull、 Pull requests(pr)

参考[Pro Git book](https://git-scm.com/book/en/v2)

###  1.2 国内连接github的三种方法

git不会走系统设置的全局代理，需要手动配置，目前有以下解决方案

1. 修改host（容易失效）
2. 设置git代理（明确自己使用的代理协议和代理地址代理端口）
3. proxychains （对macOS高版本Big Sur、Monterey不友好，需要关闭SIP，ubuntu可apt，centos需要编译安装）https://github.com/rofl0r/proxychains-ng


## 2. 编译ALICE环境
ALICE环境事实上不光是由Aliphysics软件构成的，而是类似于conda的成套环境，它包含了大量的服务于local和grid需要使用的软件包，例如JAlien

最简单最可靠的编译方法是利用官方的aliBuild工具，对于Centos7和Centos8，aliBulid可以直接由yum获得，对于macOS 可以使用homebrew，对于ubuntu等其他系统可以使用pip

~~编译始终报错可以考虑aliDock https://github.com/alidock/alidock 。aliDocker自动包含了aliBuild，但是并没有包含编译必要的软件包，依然需要手动安装依赖，速度慢，但是最稳妥，经过实测2020款Macbook Air设置4核编译成套环境需要12+小时。~~ alidock已经于今年八月弃用


```bash
mkdir -p ~/alice
cd ~/alice
#自动clone最新版的Aliphysics和依赖list
aliBuild init AliPhysics@master
#验证系统中是否已经有可以被编译拾取的依赖，将会列出大量需要重新编译的软件包，越少越好，列出的数量太多需要yum/apt update以及预先安装一些依赖
aliDoctor AliPhysics --defaults next-root6
#编译，编译过程可以加一些与make命令类似的debug参数还有编译设置例如-j8等
#重新编译将自动跳过已经编译好的包
aliBuild build AliPhysics --defaults user-next-root6
#自动清理编译生成的中间文件
#aliBuild clean

```


## 3. 编译自己的Aliphysics软件包

在github上fork ALICE的AliPhysics项目 https://github.com/alisw/AliPhysics
```bash
#替换由aliBuild init来的AliPhysics源码包
cd ~/alice
#替换为自己的github帐号
git clone git@github.com:USERNAME/AliPhysics.git
#这里应该将自己的代码加入AliPhysics
...
#重新编译AliPhysics
#最新
aliBuild build AliPhysics --defaults o2
#数月前更新，可能有报错（查看alidist）
#aliBuild build AliPhysics --defaults next-root6
#aliBuild build AliPhysics --defaults user-next-root6

#aliBuild clean
```

如果编译总是失败目前alidock依然是一个还行的选择，但是alidock自带的gcc版本过低，root与aliroot均不支持。

如果采用--defaults o2参数编译，运行root或者aliroot均会报错，一个勉强可行的方案是将需要的动态链接库（libstdc++.so.6、libgfortran.so.5等）软连接至利用aliBuild编译好的GCC-ToolChain中的动态连接库（需要root权限）。注意，重启docker一定要再运行一遍。

```bash
alidock root
#以root方式进入docker
ln -sf /persist/sw/slc7_x86-64/GCC-Toolchain/v10.2.0-alice2-6/lib64/libstdc++.so.6.0.28 /usr/lib64/libstdc++.so.6
ln -sf /persist/sw/slc7_x86-64/GCC-Toolchain/v10.2.0-alice2-6/lib64/libgfortran.so.5.0.0 /usr/lib64/libgfortran.so.5
```


## 4. 使用ALICE环境
```bash

#列出可用软件包
alienv q
#进入最新编译的环境，推荐别名ali写入shell配置
alienv enter AliPhysics/latest
#也可以不加载环境进行一些基本操作，例如
alienv setenv AliPhysics/latest -c aliroot myMacro.C+

```

当git仓库中的AliPhysics更新后，需要更新自己本地的AliPhysics时，应该进入alice文件夹下的AliPhysics文件夹运行

```bash
git pull --rebase
```
如果远端AliPhysics没有大得改动，可以运行
```bash
git pull --fetch
```
只下载新添加文件部分，原来编译好的动态连接库不会更新，可以增加编译速度。

重新编译AliPhysics不需要很长的时间，只需进入sw/BUILD文件夹make即可，运行如下命令

```bash
cd $ALIBUILD_WORK_DIR/BUILD/AliPhysics-latest/AliPhysics
alienv setenv GCC-Toolchain/latest -c cmake --build . -- -j 2 install
```
-j2 可以改成自己需要的CPU核心数目。MacBook Air 2020这种垃圾-j2就可以了，运气不好-j4都会超过温控墙直接死机。

## 5. 运行分析任务
本地与grid运行需要以下文件
- AliAnalysisTaskMyTask.cxx（分析代码）
- AliAnalysisTaskMyTask.h（分析头文件）
- AddMyTask.C（加载分析代码的脚本）
- runAnalysis.C
### 5.1 本地运行
本地运行不需要将自己的分析代码编译入AliPhysics
下载需要分析的数据集中的任意AOD文件，注意基于ROOT6和ROOT5的AliPhysics加载动态链接库代码不同

```bash
alienv enter AliPhysics/latest
aliroot runAnalysis.C
```
### 5.2 Grid运行

Grid运行不需要将自己的分析代码编译入AliPhysics
在提交Grid分析任务之前可以进行本地测试。Grid运行链接JAlien网络，需要完成如下准备工作

1. 成为ALICE会员
   alice.secretariat@cern.ch
   注册成功可以访问https://alisw.cern.ch/check

2. 生成证书
   p12证书 https://ca.cern.ch/ca/user/Request.aspx?template=EE2User

   crt证书 https://cafiles.cern.ch/cafiles/certificates/Grid.aspx
   
3. 在浏览器中注册证书

   chromium类浏览器不友好，导入成功可无锁访问https://alimonitor.cern.ch/

4. 在ALICE Grid系统注册证书

   https://voms2.cern.ch:8443/voms/alice

5. 利用Grid tools将.p12证书转化为两个验证文件

   ```bash
   
   #将.p12密钥转化为两个文件置于~/.globus
   openssl pkcs12 -clcerts -nokeys -in myCertificate.p12 -out ~/.globus/usercert.pem
   openssl pkcs12 -nocerts -in myCertificate.p12 -out ~/.globus/userkey.pem
   #注意权限设置
   chmod 0400 ~/.globus/userkey.pem
   
   ```

运行任务
```bash

alienv enter AliPhysics/latest
#alien-token-destroy
#ROOT5可在加载环境后设置token，后面多次提交任务将不需要输入PEM密码
#ROOT6版本不需要token
alien-token-init YOUR_ALIEN_USERNAME
aliroot runAnalysis.C

```

连接alien服务器
```bash

#ROOT5
aliensh
#ROOT6
alien.py

```

### 5.3 LEGO Train运行

利用LEGO Train分析数据必须要将自己的代码加入Aliphysics（参考第4章）。这种方法保证了在LEGO Train上代码不会出错，不会影响其他人的工作。参考[Analysis Trains with the LEGO Framework](https://twiki.cern.ch/twiki/bin/viewauth/ALICE/AnalysisTrains)

强烈建议在本地预先进行Train测试，测试需要如下文件

- generate.C
- env.sh
- MLTrainDefinition.cfg
- globalvariables.C
- handlers.C
- runTest.sh

也可利用此脚本下载[get_train.sh](https://twiki.cern.ch/twiki/pub/ALICE/AnalysisTrains/get_train.sh)（不推荐）

运行中将会自动生成其他文件，如lego_train.*等，另外，此网络地址http://alitrain.cern.ch/train-workdir/ 中有目前ALICE Train operator进行train测试的相关文件可供参考

值得一提的是，MLTrainDefinition.cfg中需要写明自己分析需要的依赖的分析（例如MultSelection、PhysicsSelection、PIDResponse等）。自己的分析代码内的接口也全部由本文件控制


运行测试

```bash
alienv enter AliPhysics/latest
alien-token-init
#进入测试脚本文件夹
cd myLEGO
chmod 755 runTest.sh
#运行测试
./runTest.sh
```

如果提示无法链接alien，可将generate.C中的所有TGrid相关行全部注释，并手动下载测试数据纯本地测试 http://alitrain.cern.ch/train-workdir/testdata/

测试数据需要至少两个AliAOD.root文件，它们应该被放置在不同编号的文件夹内如0001，0002

generate.C将自动生成lego_train.sh等文件

lego_train.sh以如下规则读取一个txt文件

> The name of the folders are build up out of the TEST_Dir, archive name (if [AOD](https://twiki.cern.ch/twiki/bin/view/ALICE/AOD)=2 aod_archive.zip, otherwise root_archive.zip ), FILE_PATTERN ([AliAOD](https://twiki.cern.ch/twiki/bin/edit/ALICE/AliAOD?topicparent=ALICE.AnalysisTrains;nowysiwyg=1).root) and the number of test files (2 or higher). So just take the dataset  you want to analyze, copy two of the files locally and create the a text file with the absolute paths to the test files. This text file has to  be in the same folder as runTest.sh and its name should be  x_root_archive_AliAOD_2.txt where 'x' is the string which is in env.sh  in TEST_DIR (replacing "/" with "__"). Of course if you're analyzing  aod_archive.zip, the correct name of the file is  x_aod_archive_AliAOD_2.txt

注意：该文件内需要写明本地数据的地址！如下例

```
/home/LEGO/testdata/0001/AliAOD.root
/home/LEGO/testdata/0002/AliAOD.root
```








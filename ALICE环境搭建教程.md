# ALICE环境搭建教程

Chunzheng


## 1. Git与Github

###  基概基操

参考 https://git-scm.com/book/en/v2

1. git概念、版本控制系统、github
2. 常见命令：git status、git add 、git commit、git branch、git checkout 、git merge
3. 远程仓库：git pull、 git push、git clone(http、ssh) http链接每次push需要输入帐号密码，ssh上传客户端公钥可以直接push与pull、 Pull requests(pr)

###  国内连接github的三种方法

git不会走系统设置的全局代理

1. 修改host（容易失效）
2. 设置git代理（明确自己使用的代理协议和代理地址代理端口）
3. proxychains （对Big Sur不友好，ubuntu可apt，centos需要编译安装）https://github.com/rofl0r/proxychains-ng


## 2. 编译ALICE环境
ALICE环境事实上不光是由Aliphysics软件构成的，而是类似于conda的成套环境，它包含了大量的服务于local和grid需要使用的软件包，例如JAlien

最简单最可靠的编译方法是利用官方的aliBuild工具，对于Centos7和Centos8，aliBulid可以直接由yum获得，对于macOS 可以使用Homebrew，对于ubuntu等其他系统可以利用pip

编译始终报错可以考虑alidock，alidock自动包含了aliBuild，但是并没有包含编译必要的软件包，依然需要手动安装依赖

https://alice-doc.github.io/alice-analysis-tutorial/

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
aliBuild build AliPhysics --defaults user-next-root6
#aliBuild clean
```
当AliPhysics可以在本地编译通过，即可提交pr，并联系管理人员merge

## 4. 使用ALICE环境
```bash

#列出可用软件包
alienv q
#进入最新编译的环境，推荐别名ali写入shell配置
alienv enter AliPhysics/latest
#也可以不加载环境进行一些基本操作，例如
alienv setenv AliPhysics/latest -c aliroot myMacro.C+

```

## 5. 运行分析任务
需要以下文件
- AliAnalysisTaskMyTask.cxx
- AliAnalysisTaskMyTask.h
- AddMyTask.C
- runAnalysis.C
### 5.1 本地运行
本地运行不需要将自己的分析代码编译入AliPhysics
下载需要分析的数据集中的任意AOD文件，注意基于ROOT6的AliPhysics加载动态链接库与ROOT5不同

```bash

alienv enter AliPhysics/latest
aliroot runAnalysis.C

```
### 5.2 Grid运行

本地运行不需要将自己的分析代码编译入AliPhysics
Grid运行链接JAlien网络，需要完成一些准备工作

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
   openssl pkcs12 -clcerts -nokeys -in ~/Downloads/myCertificate.p12 -out ~/.globus/usercert.pem
   openssl pkcs12 -nocerts -in ~/Downloads/myCertificate.p12 -out ~/.globus/userkey.pem
   #注意权限设置
   chmod 0400 ~/.globus/userkey.pem
   
   ```

运行任务
```bash

alienv enter AliPhysics/latest
#alien-token-destroy
#可在加载环境后设置token，后面多次提交任务将不需要输入PEM密码
alien-token-init YOUR_ALIEN_USERNAME
aliroot runAnalysis.C

```
ROOT6支持在提交Grid分析任务之前进行测试，具体设置可参考ALICE Analysis Tutorial documentation

### 5.3 LEGO Train运行

参考利用LEGO Train分析数据必须要将自己的代码加入Aliphysics（参考第4章）。这种方法保证了在LEGO Train上运行时代码在运行上不会出错，不会影响其他人的工作

...

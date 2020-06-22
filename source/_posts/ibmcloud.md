---
title: IBM Cloud Usage
date: 2020-06-22 20:25:49
tags: ibm; docker; cloud;
---


## IBM Cloud Usage

IBM Cloud 提供了256M的免费运行空间

注册地址: cloud.ibm.com

### 创建实例

Cloud Foundry 可以看作是一个docker容器实例，支持多种语言的Linux环境

1. 登录https://cloud.ibm.com/
2. 点击`Create resource`
3. 选择Cloud Foundry
4. Application Runtimes中选择自己需要的语言，目前支持Java、JS、Python、Go、Swift、PHP
5. 区域默认的`Dallas`,配置选择免费的256M；App Name输入自己应用名称，后面要用；域名选择默认的`us-south.cf.appdomain.cloud`
6. 创建完成后，会自动转到帮助页面

### Python Demo

#### code

官方提供的Demo例子，用的是Flask

`git clone https://github.com/IBM-Cloud/get-started-python`

`cd get-started-python`

#### 环境

1. 安装ibmcloud CLI程序 https://github.com/IBM-Cloud/ibm-cloud-cli-release/releases/
2. 安装Python
3. 创建虚拟Python环境 ` python -m venv pyvenv36`
4. 激活当前的虚拟环境`pyvenv36\Scripts\activate`，然后进入到下载的代码目录安装python依赖`pip install -r requirements.txt`
5. 本地执行Demo程序`python hello.py`
6. 浏览器中访问http://127.0.0.1:8000/ 可以看到一个输入框

#### 部署

安装ibmcloud CLI程序后，进入下载代码目录

1. 修改配置文件`manifest.yml`的应用名称为自己创建时写的名称如`xxxxxx`

2.  执行`ibmcloud login`登录服务，中间需要输入邮箱和密码

   ```shell
   E:\code\ibm\dev\get-started-python>ibmcloud login
   API 端點: https://cloud.ibm.com
   
   Email> xxxx@gmail.com
   
   Password>
   正在鑑別...
   确定
   
   已設定帳戶 xxxxx's Account (xxxxxxx) 的目標
   ```

   

3. 提示选择地区直接`Enter`跳过，此时会显示应用的基本信息，还会问是否给IBM统计信息，当然是no

   ```shell
   API 端點：      https://cloud.ibm.com
   地區：
   使用者：        xxxxx@gmail.com
   帳戶：          xxxx's Account (xxxxxxxxx)
   資源群組：      未設定資源群組的目標，請使用 'ibmcloud target -g RESOURCE_GROUP'
   
   CF API 端點：
   組織:
   空間：
   
   我們想要收集使用情形統計資料以協助改善 IBM Cloud CLI。
   此資料絕不會在 IBM 之外共用。
   若要進一步瞭解，請參閱 IBM 隱私權條款：https://www.ibm.com/privacy
   您可以啟用或停用使用情形資料收集，方法是執行 'ibmcloud config --usage-stats-coll
   ect [true | false]'
   
   您要傳送使用情形統計資料給 IBM 嗎？ [y/n]> n
   ```

   

4. 选择要用的cf应用节点`ibmcloud target --cf`，这个过程需要代理，否则可能会提示网络错误

   ```shell
   失败
   無法取得 Cloud Foundry 實例：
   Get "https://mccp.us-south.cf.cloud.ibm.com/v2/regions": dial tcp: lookup mccp.u
   s-south.cf.cloud.ibm.com: no such host
   ```

   **正常的输出**

   ```shell
   E:\code\ibm\dev\get-started-python>ibmcloud target --cf
   
   選取 Cloud Foundry 實例：
   1. public CF us-south (https://api.us-south.cf.cloud.ibm.com)
   2. public CF eu-de (https://api.eu-de.cf.cloud.ibm.com)
   3. public CF eu-gb (https://api.eu-gb.cf.cloud.ibm.com)
   4. public CF au-syd (https://api.au-syd.cf.cloud.ibm.com)
   5. public CF us-east (https://api.us-east.cf.cloud.ibm.com)
   請輸入數字> 1
   目標 Cloud Foundry (https://api.us-south.cf.cloud.ibm.com)
   
   已設定組織 xxxx 的目標
   
   已設定空間 dev 的目標
   
   API 端點：      https://cloud.ibm.com
   地區：
   使用者：        xxxxx@gmail.com
   帳戶：          xxxxxx's Account (xxxxxxxxx)
   資源群組：      未設定資源群組的目標，請使用 'ibmcloud target -g RESOURCE_GROUP'
   
   CF API 端點：   https://api.us-south.cf.cloud.ibm.com（API 版本：2.148.0）
   組織:           xxx
   空間：          xxx
   ```

   *其中的组织和空间都可以通过网站的账户下面更改名称，免费账户只能有一个组织*

5. 安装Cloud Foundry CLI `ibmcloud cf install`

6. 本地代码push到服务器`ibmcloud cf push` 会输出一堆日志和部署信息，最终会显示系统的运行信息

   ```shell
   正在等待應用程式啟動...
   
   名稱:            xxxxx
   所要求的狀態:    started
   路徑:            xxxxxx.us-south.cf.appdomain.cloud
   前次上傳:        Mon 22 Jun 22:43:39 CST 2020
   堆疊:            cflinuxfs3
   建置套件:        python
   
   類型:         web
   實例:         1/1
   記憶體用量:   128M
   啟動指令:     python hello.py
        state    自從                   cpu    memory       磁碟        詳細資料
   #0   執行中   2020-06-22T14:44:05Z   0.4%   18.8M/128M   198.7M/1G
   
   ```

   

7. 浏览器访问`xxxxxx.us-south.cf.appdomain.cloud`就可以看到应用

8. 使用`ibmcloud cf ssh appname`可以以ssh访问应用的容器空间，不过我试了一直提示`no such host`



*先到这里，休息一下*


---
## iOS基于Jenkins+fastlane自动化打包发布

#### 一、登录

首先需要登录Jenkins服务器页面，登录地址：http://192.168.116.252:8080




登录账户：shengweijie

登录密码：ule.com

### 二、构建工程

#### 1、选择需要构建的工程

![](/Users/chenzhuqing/Desktop/Mardown/Resource/2.png)

#### 2、点击配置，修改shell脚本进行构建不同的ipa包，主要包括debug测试包，AdHoc测试包，AppStore发布版（AppStore版本则会发布到AppStore中，其他版本则会发布到蒲公英中）


![](/Users/chenzhuqing/Desktop/Mardown/Resource/3.png)

修改Shell脚本进行构建不同版本的APP

![](/Users/chenzhuqing/Desktop/Mardown/Resource/4.png)

	
	#如果需要更新本地spec文件的时候，就需要调用pod update
	pod update 
	#如果没有需要更新的spec文件，则可以直接用pod install 命令
	#pod install
	#调用不同的命令进行构建
	fastlane ios uleDebug

fastlane ios uleDebug : 构建debug 版本。用于测试

fastlane ios uleRelease: 构建AdHoc版本

fastlane ios uleAppStore:构建AppStore版本。

#### 3 Debug 和 Adhoc 版本添加安装二维码

为了能够在上传ipa之后下载二维码图片，我们可以采用jenkins的pgyer的插件，安装好插件之后 在构建执行Sell命令后面，继续添加Upload to pgyer with apiV1. 具体如下所示：

![](/Users/chenzhuqing/Desktop/Mardown/Resource/5.png)

需要注意的是：

qrCodePath 和 envVarsPath 的值就采用蒲公英官网上提供的固定的值就可以。

安装build Description 插件 该插件是用来显示安装二维码。

安装成功后

要显示二维码还需要要Jenkins的：全局安全配置中-->标记格式器-->选择Safe HTML 如下图所示：

![](/Users/chenzhuqing/Desktop/Mardown/Resource/7.png)

然后在构建中添加Set build descriptioon ，具体填写值如下所示：

![](/Users/chenzhuqing/Desktop/Mardown/Resource/6.png)

至此，就可以对工程进行直接构建，构建成功后就能显示一个安装二维码，点击二维码还能跳转到下载页面：

![](/Users/chenzhuqing/Desktop/Mardown/Resource/8.png)
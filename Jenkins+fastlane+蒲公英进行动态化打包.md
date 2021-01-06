## Jenkins+fastlane+蒲公英进行动态化打包

### 一、Jenkins 的安装配置

#### 1、java JDK 安装
Jenkins 安装前首先需要安装java JDK，目前jenkins官方要求必须安装java 8 - java11。因此我们可以到java官方网站下载并安装好java SDK。

#### 2、jenkins安装

到官网 <https://www.jenkins.io/download/lts/macos/>

确认是否已经安装过homebrew，如果没有安装，则需要先安装homebrew。在终端命令行安装如下：

	/usr/bin/ruby -e "$(curl -fsSL https://cdn.jsdelivr.net/gh/ineo6/homebrew-install/install)"

mac系统中可以通过homebrew 来进行命令行方式安装。

	brew install jenkins-lts
	
安装好后面，需要启动

	brew services start jenkins-lts
	
如果是重新启动需要调用

	brew services restart jenkins-lts

启动完成后，在浏览器中输入 <http://localhost:8080>

选择左侧的选项，安装Jenkins推荐的插件即可

![图片2](/Users/chenzhuqing/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/249261843/QQ/Temp.db/CEC7636E-C750-4158-A8E3-051FCFD090C4.png)

安装完成后就可以注册一个管理者账号进行登录了。

![图片3](/Users/chenzhuqing/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/249261843/QQ/Temp.db/34670474-12E9-40FC-8C2C-D8F744339EB0.png)

####3、配置Jenkins

在配置Jenkins 环境变量的时候，需要注意的是cocoapds 和 fastlane 的环境变量需要添加的配置中。

![图片4](/Users/chenzhuqing/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/249261843/QQ/Temp.db/DE6E4BFB-0B06-474E-A31D-79306CD92CCF.png)

如何获取图中的PATH路径呢，

首先我们需要在系统的终端打开.bash_profile 文件，
	
	cd ~
	open .bash_profile
	
并添加如下变量：

	export PATH="$PATH:$HOME/.rvm/bin"
	export PATH="$HOME/.fastlane/bin:$PATH"

然后再终端输入：

	echo $PATH

便能在终端打印出PATH的值，将其填入的图中的PATH对应的值的地方就可以了。

至此，我们再打开Jenkins的Shell脚本构建，就可以在其中执行pod install 以及fastlane等命令了。

在shell脚本中需要添加

	export LANG=en_US.UTF-8
	export LANGUAGE=en_US.UTF-8
	export LC_ALL=en_US.UTF-8

如果没有再shell脚本中添加以上代码，在执行脚本的时候可能会出现编码UTF-8的错误。

### 4、jenkins mac系统中局域网访问配置

使用brew安装jenkins会避免很多其他安装方式产生的用户权限问题，但是会将httpListenAddress默认设置为127.0.0.1，这样我们虽然可以在本地用localhost:8080访问，但是本机和局域网均无法用ip访问。解决办法为修改两个路径下的plist配置。并重启

	～/Library/LaunchAgents/homebrew.mxcl.jenkins.plist

	/usr/local/opt/jenkins/homebrew.mxcl.jenkins.plist

将上面两个plist中的httpListenAddress后的ip地址，修改为本机IP或者0.0.0.0即可。


###  二、Fastlane 安装配置

#### 1、安装
到官网安装fastlane<http://docs.fastlane.tools/getting-started/ios/setup/>

	sudo gem install fastlane -NV
	brew install fastlane

#### 2、初始化

安装好fastlane之后，就需要在终端执行初始化fastlane命令，

首先打开终端，cd到工程文件目录下：

然后调用命令行：

	fastlane init

这会出现以下选择项：

![](/Users/chenzhuqing/Desktop/Mardown/Resource/10.png)

选择3，可以自动生成DeliverFile文件，如果选择其他的则不会自动生成，如果已经初始化了，但是么有deliverFile文件，我们可以在当前目录下调用命令行：fastlane deliver init 生成一个。

最后生成的文件目录如下所示。

![](/Users/chenzhuqing/Desktop/Mardown/Resource/11.png)

#### 3、脚本编写
打开Fastfile 文件编写编译打包上传蒲公英的脚本如下所示：
	
	
	APP_NAME = "UleLiveApp"
	WORKSPACE = "UleLiveApp.xcworkspace"
	SCHEME = "UleLiveApp"
	IPA_TIME = Time.now.strftime("%Y%m%d_%H%M")
	OUTPUT_DIRECTORY = "packages"
	ENV_PREFIX=""
	IPA_NAME = ""
	#APIKEY = "fd0de8948ea19f320acc7590ed3f4418"
	#USERKEY = "37d03c8be3dd48a1f8014a2d182da740"
	
	platform :ios do
	  
	  before_all do
	       xcode_select "/Applications/Xcode.app"
	       FASTLANE_XCODEBUILD_SETTINGS_TIMEOUT = "40"
	  end
	
	  #debug包
	  lane :uleDebug do
	    ENV_PREFIX="debug_"
	    EXPORT_METHOD = "development"
	    match(type:"development")
	    package(configuration: "Debug")
	    #pgyer(api_key: "#{APIKEY}", user_key: "#{USERKEY}", password: "123456", install_type: "2")
	  end
	
	  #release包
	  lane :uleRelease do
	    ENV_PREFIX="adhoc_"
	    EXPORT_METHOD = "ad-hoc"
	    match(type:"adhoc")
	    package(configuration: "Release")
	    #pgyer(api_key: "#{APIKEY}", user_key: "#{USERKEY}", password: "123456", install_type: "2")
	  end
	
	  lane :uleTesting do
	    ENV_PREFIX="testing_"
	    EXPORT_METHOD = "ad-hoc"
	    match(type:"adhoc")
	    package(configuration: "Testing")
	    #pgyer(api_key: "#{APIKEY}", user_key: "#{USERKEY}", password: "123456", install_type: "2")
	  end
	
	
	    #发布包
	  lane :uleAppStore do
	    ENV_PREFIX="appstore_"
	    EXPORT_METHOD = "app-store"
	    match(type:"appstore")
	    #自动增加build 版本号
	    increment_build_number
	    package(configuration: "Release")
	    #上传AppStore
	    deliver(
	        force:true,
	        #app_version:"1.0.2",
	        automatic_release:false,
	        ipa:"#{OUTPUT_DIRECTORY}/#{IPA_NAME}"
	    )
	  end
	
	  #打包函数
	  lane :package do |option|
	      #cocoapods
	      PLIST_INFO_VERSION = get_version_number(target: "#{SCHEME}")
	      IPA_NAME = "#{ENV_PREFIX}" + "#{APP_NAME}_"  +  "#{IPA_TIME}_" + "#{PLIST_INFO_VERSION}" + ".ipa"
	
	     #打包
	     gym(
	      scheme:  "#{SCHEME}",
	      export_method: "#{EXPORT_METHOD}",
	      configuration: option[:configuration],
	      output_directory: "#{OUTPUT_DIRECTORY}",
	      include_symbols: true,
	      include_bitcode: false,
	      xcargs: 'DEBUG_INFORMATION_FORMAT="dwarf-with-dsym"',
	      output_name: "#{IPA_NAME}",
	      export_xcargs: "-allowProvisioningUpdates"
	      )
	      xcclean(
	       workspace: "#{WORKSPACE}",
	       scheme: "#{SCHEME}"
	      )
	   end
	      #打包上传成功后
	   after_all do |lane|
	
	   end
	   #出现bug
	   error do |lane, exception|
	
	   end
	end

注意：increment_build_number 这个会自动增加build版本，

### 三、蒲公英插件安装

	fastlane add_plugin pgyer
	
安装过程中报错：

	There was an error while trying to write to
	`/Users/chenzhuqing/.bundle/cache/compact_index/rubygems.org.443.29b0360b937aa4d161703e6160654e47/versions`.
	It is likely that you need to grant write permissions for that path.

看日志信息是权限问题，因此尝试修改.bundle 目录的权限，如下：

	sudo chmod -R 777 /Users/chenzhuqing/.bundle/

然后再次安装，成功安装蒲公英插件

### 四、通过match进行证书管理

#### 1、初始化match

为了能够是git管理证书文件，首先我们需要在git服务器上建一个保存证书的仓库。

然后在工程文件目录中（与fastlane 同一级目录）调用命令行：

	fastlane match init
	
此时终端会提示需要输入管理证书的git地址，我们公司目前使用的gitlab管理证书，因此我们提供一个gitlab的地址如下：

	http://git.uletm.com/mobile-app-library/ios/certificates.git
	
完成之后，这会在fastlane目录下，生成一个Matchfile目录

然后，打开Matchfile文件修改其中的的app_identifier 和username

	app_identifier(["com.appule.umarket", "com.appule.umarket.usNotificationServer"])
	username("uleapp@tom.com") # Your Apple Developer Portal username

写两个就会生成两个对应的profile文件。如果工程中有推送没有写推送的bundleID，就不会自动生成推送的profile文件。

##### 2、用match命令生成证书以及profile

	fastlane match adhoc
	fastlane match development
	fastlane match addstore

如果AppStore中证书已经存在了。并且超过3个证书，那么就必须要先删除已经存在的证书然后再调用上面的命令自动生成新的证书。删除旧的证书不会影响已经发布或者正在审核的App。

	fastlane match adhoc --readonly
	fastlane match development --readonly
	fastlane match addstore --readonly

如果执行过程中，出现输入Git Repo密码后，密码错误导致的不能解密repo，可以尝试着用fastlane match change_password来重置密码。

#### 3、如何自动增加build号

首先需要调用increment_build_number

修改buildsettings里面的version配置，current project version 随便填一个。versionsystem 选择apple generic。

    #自动增加build 版本号
    increment_build_number({
       build_number: latest_testflight_build_number.to_i + 1
    })
 
 
#### 4、工程无法自动匹配证书和描述文件

在实际操作过程中，我们发现在打包机上，通过jenkins 下载的工程，编译时，fastlane无法正常匹配证书和描述文件，而这两个文件通过match方法，是已经成功下载并安装到本地了。

最后我们只能将自动模式，改成手动选择。
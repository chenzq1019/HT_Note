## cocoapods 在SVN环境中的私有pods建立过程

#### 一、 安装repo-svn 插件
repo-svn插件的作用用来建立并上传spec文件的，安装的命令：

    gem install cocoapods-repo-svn
    
repo-svn 其他命令

* 添加私有svn 的spec到项目库中

		pod repo-svn add SpecRepo http://chenzhuqing@svn.tomshopping.com/shanghai/mobileclient/iPhone/SpecRepo
	
这个命令会在本地 ~/.cocoapods/repos/ 目录中添加一个SpecRepo，后面所有的私有pod的spec文件都会在该文件中，
	注意SpecRepo需要事先在svn的服务器中创建好，然后再调用上面的命令。
	
* 将spec文件提交远端的sepc库中。
  
  
 		pod repo-svn push SpecRepo FirstPod.podspec 
	

注意调用该命令，需要cd到podspec文件目录中，例如：FirstPod.podspec所在的目录。同时需要确保文件没有错误。验证通过就可以。

	 pod repo-svn lint FirstPod.podspec
	 
push命令结束后，会把本地的podspec文件上传的svn的SpecRepo的仓库中。


	pod repo-svn update my-svn-repo 更新项目
	
	pod repo-svn remove my-svn-repo 删除项目

#### 二、 创建私有pod

1、利用命令：

	pod lib create  FirstPod
这个命令可以创建一个本地pod工程，
此时podfile文件中，会将class文件中的所有文件作为本地pod导入到工程中。

    //podfile 文件中本地pod的写法。（这个会自动生成）
    pod ‘FirstPod’, :path=>’../'
 
 2、可以将需要组件化的文件都放在class文件中
 
 3、修改podspec文件
 
 4、调用验证podspec


	pod repo-svn lint FirstPod.podspec
	
 5、将工程上传的svn服务器中。
 
 6、将podspec导入到svn的spec仓库中，利用之前的命令
 
 	pod repo-svn push SpecRepo FirstPod.podspec
 	
#### 三、使用私有pod

1、 在工程的podfile文件中，使用插件指定数据源地址，然后就可以像公用的pod一样使用了。如果不指定源，就需要每个私有pod都要具体写上svn对应的地址。

	source 'https://github.com/CocoaPods/Specs.git'
	target 'TestDemo' do
    plugin 'cocoapods-repo-svn', :sources => [
    'http://svn.tomshopping.com/shanghai/mobileclient/iPhone/SpecRepo' # 添加 svn 服务器中私有库 spec 的 repo
    ]
	pod 'FirstPod'
	pod 'UleMediator'
	end
	
不用插件来指定源的方式：	

	target 'TestDemo' do
	pod 'UleMediator', :svn =>"http://svn.tomshopping.com/shanghai/mobileclient/iPhone/UleCodeLibrary/UleMediator/",:tag =>"1.0.0"
	end

#### 四，查询私有库

用命令： pod search podName

如果查不到，可以先执行 rm ~/Library/Caches/CocoaPods/search_index.json 再次查询
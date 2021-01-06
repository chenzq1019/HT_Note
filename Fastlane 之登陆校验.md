## Fastlane 之登陆校验

如果账号没有开启两步验证的话,到这里,整个打包过程就结束了.但是作为开发者的我们,经历过去年 apple的一轮协议更新,主账号一般都开启了两步验证了,因此我们非常悲催的发现一个场景,在链接apple connect的时候,它报错了

### 1、获取专有密码
登录账号管理获取app专用密码

appleID 账号管理中，安全->获取专用密码

### 2、编辑zshrc文件，有些是编辑bash_profile文件

使用open 命令打开文件
 在文件中添加全局变量
 
 	# 添加全局变量
	export FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD=App专用密码

### 3、获取FASTLANE_SESSION

使用fastlane 命令在终端输入以下命令

	fastlane spaceauth -u 你的appID

然后便能获取到fastlane_session 的值

![](/Users/chenzhuqing/Desktop/Mardown/Resource/9.png)

### 4、再次编辑zshrc或者bash_profile文件
 在文件中添加FASTLANE_SESSION
 
 
### 5、在jenkins的环境变量中添加我们在bash_profile中的值。
最后，我们还需要在jenkins中添加一下环境变量。

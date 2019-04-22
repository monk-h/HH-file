#iOS项目持续集成
##（Jenkins+svn+cocoapods+蒲公英+shell修改环境等相关变量）

最近安装部署Jenkins环境打包iOS，踩了无数坑，记录一下安装步骤。mac环境macOS Mojave 10.14.4
###第一步：安装Jenkins
因为Jenkins是基于java的，所以先安装JDK，在安装Jenkins，网上很多教程，不在描述。我安装的Jenkins ver. 2.164.2
###第二步：新建Jenkins任务
* 点击新建任务 

![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/xj.jpg)

* 输入一个任务名称，点击构建一个自由风格的软件项目
![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/creatName.png)

* 然后进入任务配置
	* 第一部分： General
		* 描述随意，丢弃旧的构建设置如下
		![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/general.jpg)
		* 参数化构建过程：通过参数配置，可构建指定环境，版本等相关设置的包
			* 点击添加参数
			![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/general2.jpg)
			* 参数设置如下，我设置了环境变量，app版本，build版本，编译环境debug/release;可根据自己项目需要进行相关设置
			![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/general3.jpg)
			![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/general4.jpg)
	* 第二部分：源码管理
		公司使用的svn，所以我也设置的配置svn
		![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/ymgl.jpg)
	* 第三部分：没有用到触发器，暂不设置
	* 第四部分：构建环境，无勾选。
	* 第五部分：构建；（重点来了，主要设置内容）
		* 点击增加构建步骤,选择执行shell，
		![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/gj.png)
		* 增加第一个执行shell脚本：项目里用到了cocoapods，所以构建先执行脚本pod 
		![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/gj2.jpg)
		
			``` 
			#bin/bsah - l
			export LANG=en_US.UTF-8
			export LANGUAGE=en_US.UTF-8
			export LC_ALL=en_US.UTF-8
			cd $WORKSPACE/你的构建项目名称/
			/usr/local/bin/pod install --verbose --no-repo-update
			```
		`注意：不增加export为utf-8容易报错`
		* 增加第二个执行shell脚本：使用xcodebuild命令进行打包签名和输出ipa包。
		![avatar](https://raw.githubusercontent.com/monk-h/HH-file/e3d8e54ce34d756bd50640c3ea3bc0647f816f58/img/gj3.jpg)
		我的项目环境是用plist加载的，所以我使用脚本，修改了plist里的环境变量值
		
			```
			#工程名字(Target名字)
			Project_Name="工程名字"
			#workspace的名字
			Workspace_Name="workspace的名字"
			
			#配置环境，设置参数
			if [ ${Environment} == "你之前设置的构建参数值（测试环境1）" ];then
			环境变量名字=测试环境1
			else
			环境变量名字=测试环境2
			fi
			
			TARGET_APP_VERSION=${AppVersion}
			TARGET_CFBundleVersion=${BundleVersion}
			
			basepath=$(cd `dirname $0`; pwd)
			
			#修改plist里环境变量，版本号，build号
			/usr/libexec/PlistBuddy -c "Set 环境变量名字 $TARGET_WEB_DOME_GATEWAY" "$WORKSPACE/$Project_Name/"存储环境变量的.plist
			/usr/libexec/PlistBuddy -c "Set CFBundleShortVersionString $TARGET_APP_VERSION" "$WORKSPACE/$Project_Name/"存储版本号的.plist
			/usr/libexec/PlistBuddy -c "Set CFBundleVersion $TARGET_CFBundleVersion" "$WORKSPACE/$Project_Name/"存储build号的.plist
			
			#Release或者Debug
			Configuration=${Configuration}
			
			EnterpriseBundleID="com.***.***"
			
			#企业(enterprise)证书名
			#描述文件
			ENTERPRISECODE_SIGN_IDENTITY="iPhone Distribution: 名称"
			ENTERPRISEROVISIONING_PROFILE_NAME="********-****-****-****-************"
			#ExportOptions.plist这个文件是自己用xcode传统打包渠道打出来后得到文件，去导出的包里寻找吧
			EnterpriseExportOptionsPlist=./ExportOptions.plist
			
			#企业打包脚本
			xcodebuild -workspace $Workspace_Name.xcworkspace -scheme $Project_Name -configuration $Configuration -archivePath build/$Project_Name_enterprise.xcarchive archive build CODE_SIGN_IDENTITY="${ENTERPRISECODE_SIGN_IDENTITY}" PROVISIONING_PROFILE="${ENTERPRISEROVISIONING_PROFILE_NAME}" PRODUCT_BUNDLE_IDENTIFIER="${EnterpriseBundleID}"
			xcodebuild -exportArchive -archivePath build/$Project_Name_enterprise.xcarchive -exportOptionsPlist ${EnterpriseExportOptionsPlist} -exportPath $WORKSPACE/build/$Project_Name_enterprise.ipa
	```
		<mark>关于ExportOptions.plist，这货哪来的，自己用xcode传统打包方式打出来后得到文件，去导出的包里寻找吧，找到后放到jenkins的workspace里的项目目录下就好`/User/Share/Jenkins/Home/workspace/项目名称`,没有目录就在配置不完整的情况下构建一次，会生成目录，不嫌麻烦想自己创建也ok</mark>
		* 增加第三个执行shell脚本：上传蒲公英
		 
			```
			curl -F "file=@$WORKSPACE/build/$Project_Name_enterprise.ipa/$Project_Name.ipa" -F "uKey= 去蒲公英账号里找" -F "_api_key=去蒲公英账号里找" https://qiniu-storage.pgyer.com/apiv1/app/upload
			```
			jenkins到这里就设置结束啦~~~~~
			
			<mark>证书证书证书，现在直接构建会出现证书找不到的问题，去钥匙串里把登录里的证书拽到系统里，这样jenkins就有权限使用了~~~~去构建吧🤪</mark>

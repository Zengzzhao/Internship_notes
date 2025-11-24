# 按键

聚焦搜索：`command 空格`

全屏截屏：`command shift 3`

选区截屏：`command shfit 4`

多种截屏、录屏自由选择：`command shift 5`

浏览器放大/缩小：`command +/-`

鼠标右键：`双指轻按触控板/control 单击触控板`

系统自带输入法情况下，中英文切换：`轻按capslock`

系统自带输入法情况下，切换大小写：`长按capslock知道绿灯亮/熄灭`

单个字母大写：`shift 单个字母`

切换输入法：`fn/control 空格`，切换了输入法后大小写切换为`capslock`，中英文切换为`shift`

显示/隐藏.开头的隐藏文件：`control shift .`

强制退出应用程序：`option command esc`

刷新页面：`command R`

强制刷新页面：`command shift R`

## VSCode中的快捷键

向下复制当前行：`shift option ↓`

向上复制当前行：`shift option ↑`



# 触控板

缩放：双指捏和

切换全屏显示的应用程序：三指左右轻扫

打开应用启动台：捏拢大拇指与其他三个指头

退出应用启动台：张开大拇指与其他三个指头



## vscode格式化代码

`shfit option F`：格式化代码



# 文件系统

```
/
	--Applications(应用文件夹，即我们安装应用时拖动到到文件夹，该文件夹中的应用都会显示在启动台中)
	--dev          
	--Library      
	--sbin         
	--Users
  		--Shared
  		--zengzizhao(username),用户主目录,简写为～
  				--Downloads        
  				--Music
					--Desktop(桌面)       
					--Library          
					--Pictures
					--Documents        
					--Movies           
					--Public
	--Volumes
	--bin          
	--etc          
	--opt          
	--System       
	--usr
  --cores        
  --home         
  --private      
  --tmp          
  --var
```

用户主目录$Home为/User/yourusername，简写为～。Desktop桌面、Downloads下载、Music音乐等都在用户主目录下



# 命令行解释器Shell

Mac默认shell为zsh，让用户输入命令来和操作系统进行交互

bash（Bourne-Again SHell，Linux的基础Shell）、zsh（Z Shell，bash的增强版）为Linux/Unix系的Shell

cmd（Common Prompt，Windows的传统命令后，源于DOS系统）、powershell（微软推出的跨平台Shell）为Windows系的Shell

## .zshrc文件

.zshrc是mac上Z Shell的用户级运行时配置文件，当启动zsh时会自动执行这个文件里的命令，设置好命令行环境。（该文件的内容包含了系统环境变量，类似windows的系统环境变量）





# 应用下载

[破解软件下载网站](https://xclient.info/)

## 打不开xxx，因为来自身份不明的开发者：解决

原因：在MAC下安装一些软件时提示"来自身份不明开发者"，这是MAC启用了安全机制。默认只信任Mac App Store下载的软件和拥有开发者 ID 签名的应用程序。

**解决办法**

方法1：系统设置中的隐私与安全性选项卡最下面的安全性设置当前软件信任

方法2：终端输入`sudo spctl --master-disable`、电脑解锁密码后回到系统设置中的隐私与安全性选项卡最下面的安全性设置中的允许以下来源的应用程序选择任何来源



# 新机开荒

参考博文：[链接](https://zhuanlan.zhihu.com/p/48207191)

## homebrew

homebrew是mac上的软件包管理器（类似于App Store，不过是通过命令行来安装软件而非图形化界面），主要作用是更加方便轻松安装、更新、卸载在App Store中没有的软件。

有了homebrew后，在终端中使用brew insatll xxx命令，homebrew就会自动下载、编译、安装好软件。而不需要再去官网下载、解压、拖拽到application文件夹中了。

## Iterm2

Iterm2是mac自带的Terminal终端应用程序的增强版（支持一个窗口开多个终端、一个窗口分成多个窗格），底层调用的仍然是mac自带的默认zsh

新建终端tab页：`command t`

水平分屏：`command d`

垂直分屏：`command shift d`

## on-my-zsh

zsh是mac的默认shell，而on-my-zsh是社区维护的一个zsh配置框架，包含了很多好看的主题、好用的插件、方便的工具。安装后即可获得好看的命令提示符

## nrm

nrm是npm源管理工具，可以方便地在不同npm源之间切换

```bash
# 查看当前所有npm源
nrm ls
# 添加npm源
npm add npm源名 npm源
# 使用某npm源
npm use npm源名
```



## 非开发其他有用工具

### 超级右键

[官网](https://www.better365.cn/irightmousepro.html)

Mac新建文件需要先打开软件再新建，与Win不同可以直接通过右键新建。安装了超级右键后即可使Mac具有和Win一样的右键功能




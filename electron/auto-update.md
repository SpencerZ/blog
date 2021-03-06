# electron 自动更新

> 使用electron-builder和electron-updater实现windows和mac下的自动更新功能

## 安装
```
npm i electron-builder
npm i electron-updater
```
当前安装版本：
electron-builder: 22.8.1
electron-updater: 4.3.5

## 添加事件监听
- 需要在主进程文件（main.ts）内注册Auto update事件，我是在createWindow内，也就是创建了窗口后，添加了事件监听。
- [UpdateInfo](https://www.electron.build/auto-update#updateinfo)
```ts
const mainWindow = new BrowserWindow({...});
updateHandler();// 事件监听包含在这个方法内
/**
 * 在你设定的时间点触发检查更新
 */
autoUpdater.checkForUpdates();
```
```ts
updateHandler() {
	/**
	 * 设置安装包和.yml文件存放的url，autoUpdater通过检查feedUrl下的
	 * latest.yml或latest-mac.yml获取最新版本信息
	 */
	autoUpdater.setFeedURL(uploadUrl);
	
	/**
	 * autoUpdater在上面方法中获取到了最新版本，可在此事件中获取到最新版本的信
	 * 息，包含版本号，发布内容，发布时间等
	 */
	autoUpdater.on('update-available', function (info: UpdateInfo) {
		sendUpdateMessage(info);
	});
	
	/**
	 * autoUpdater在获取更新到下载更新过程中，如果有报错，就会发出事件
	 */
	autoUpdater.on('error', function (error: Error) {
		console.log(error);
  	});
  
  	/**
   	 * autoUpdater如果没有获取到最新版本，发出事件
   	 */
  	autoUpdater.on('update-not-available', function (info) {
    		sendUpdateMessage(null);
	});
	
	/**
	 * autoUpdater在更新下载完成后，发出事件
	 */
	autoUpdater.on('update-downloaded', function (info) {

		/**
		 * 主进程监听开始更新通知，执行安装更新方法
		 */
		ipcMain.on(MainAction.UPDATE_NOW, (e, arg) => {
			console.log("开始更新");
			autoUpdater.quitAndInstall();
			/**
			* 在执行安装更新过程中，如果不加app.exit()方法，会在安装到一半的时候突然退出，
			* 并且没有任何报错
			* 原因应该是此时程序并没有真正退出，导致安装失败，此时可以在/Users/Administrater/AppData/Local/client-update文件夹内看到有pending文件夹内，存放了新版本的安装包
			*/
			app.exit();
		});
	});
}

// 通过main进程发送事件给renderer进程，提示更新信息
function sendUpdateMessage(info) {
  mainWindow.webContents.send(MainAction.UPDATE_INFO, info)
}
```
- 在渲染进程监听事件
```ts
// 监听新版本信息
ipcRender.on(MainAction.UPDATE_INFO, (event, info: UpdateInfo) => {
  console.log(info);
});
		
// 向主进程发送事件，请求更新
ipcRenderer.send(MainAction.UPDATE_NOW);
```

至此，Auto Updater的更新事件监听完成

## 配置
在electron-builder打包时，要添加publish, releaseInfo配置，在打包的时候配置更新发布url和新版本的更新内容
注意： publish.url需要跟上述的feedUrl一致。
```json
{
	"build": {
		"publish": {
			"provider": "generic",
			"url": "http://"
		},
		"releaseInfo": {
			"releaseNotes": "当前发布版本的更新内容，可在上述updateInfo.releaseNotes得到"
		}
	}
}
```

## Mac代码签名
在打包Mac包时，需要加上签名，在XCode中，从Preferences中进入 => 添加Apple ID账号 =》Manage Certificates =》 添加证书即可。

## 版本号
修改版本号时，可能会有不同的需求，需要加alpha或beta版本，这个时候打包后会生成alpha.yml或者beta.yml。
### 正式版本
version: '1.0.0'
生成latest.yml，在检查更新的时候会自动获取latest.yml，可以只更新latest.yml来发布正式版本

### beta版本
version: '1.0.0-beta.1'
生成beta.yml，在检查更新的时候也会自动获取latest.yml，但是如果我只想它检查beta.yml，这个时候需要去设置channel
```ts
autoUpdater.channel = 'beta';
```

### 如何区分正式和测试环境
1. 正式用户使用的是正式版本，检查的是latest.yml，所以只需要更新latest.yml就可以实现正式用户的应用更新。
2. 测试用户使用的是beta版本，检查的是beta.yml，所以只需要更新beta.yml就可以实现测试用户的应用更新。

### 规定应用检查更新channel
```ts
autoUpdater.channel = 'latest';// 默认值，只检查latest.yml或者latest-mac.yml
autoUpdater.channel = 'beta';// 检查beta和latest
autoUpdater.channel = 'alpha';// 检查alpha, beta, 和latest
```

## 打包
执行windows和Mac打包命令后，将得到的文件上传到feedUrl下。

- Mac
	* latest-mac.yml
	* **-mac.zip
	* **.dmg
- Windows
	* latest.yml
	* **.exe

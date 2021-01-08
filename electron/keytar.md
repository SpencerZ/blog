# 记住密码

> 使用keytar实现，将用户及密码保存在系统keychain内

如何安全地保存用户名及密码，使用老办法用文件保存不安全，并且还要考虑加密解决，而Window和Mac都有自己的密码管理器，Mac的KeyChain，Windows的凭据管理器。

> 凭据管理器打开方式：控制面板 => 用户账号 => 凭据管理器，app的密码保存在Windows凭据下。

那我们使用什么来将密码保存进密码管理器呢？**Node.js**提供了[**C++　addons**](https://nodejs.org/api/addons.html)，它提供了在javascript和C/C++库之间的接口，通过**N-API**，我们可以间接操作C/C++库，实现到密码管理器的操作。

## keytar
[**keytar**](https://github.com/atom/node-keytar)是一个可以获取、添加、删除系统管理器内密码的Node模块。

### 安装
```
npm install keytar
```

### 使用
keytar的方法返回的都是promise，所以需要在回调中执行后续方法。
service即应用唯一名称。

```ts
// 设置账号及密码，如果账号不存在，就添加密码，若存在，就更新密码
keytar.setPassword(service, account, password);

// 获取账号及密码
keytar.getPassword(service, account);

// 删除账号及密码
keytar.deletePassword(service, account);

// 获取服务下的所有账号及密码
keytar.findCredentials(service);
//返回 [{account: 'foo', password: 'bar'}]
```

**使用前建议**: 调用keytar方法尽量都在主进程文件内，避免系统弹出权限检查要求输入密码弹框，优化体验。

```ts
/**
 * 记住密码，从渲染进程发出
 */
ipcMain.on('REMEMBER_PASSWORD', (e, loginAccount) => {
	const {account, password} = loginAccount;
	keytar.setPassword(CLIENT_NAME, account, password);
});

/**
 * 在窗口初始化后，由渲染进程发出，获取最近一次的登录账号
 */
ipcMain.on('GET_RESTORED_ACCOUNT', () => {
	keytar.findCredentials(CLIENT_NAME).then(account => {
		const lastAccount = [...account].pop();
		mainWindow.webContents.send('SET_LOGIN_ACCOUNT', lastAccount);
	});
});
```
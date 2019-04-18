<!-- TOC -->

- [1. UserAgent检测](#1-useragent检测)
- [2. Webdriver检测](#2-webdriver检测)
- [3. chrome属性检测](#3-chrome属性检测)
- [4. Permissions检测](#4-permissions检测)
- [5. Plugins长度检测](#5-plugins长度检测)
- [6. The Languages检测](#6-the-languages检测)

<!-- /TOC -->
# 1. UserAgent检测
无头模式下的UA会带有HeadlessChrome关键字，例如Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/70.0.3521.2 Safari/537.361，因此可以检查UA中的关键字。
```js
if (/HeadlessChrome/.test(navigator.userAgent)) {
  // headless...
}
//修改UserAgent即可绕过
```
# 2. Webdriver检测
无头模式下navigator.webdriver为true，因此可以进行如下检测。
```js
// Webdriver Test
if (navigator.webdriver) {
  // headless...
}
//重新设置该属性即可绕过
Object.defineProperty(navigator, 'webdriver', {
    get: () => false,
  });
```
# 3. chrome属性检测
在无头模式下window.chrome属性是undefined，而在正常有界面模式下，定义如下。
```js
csi: ƒ ()
embeddedSearch: {searchBox: {…}, newTabPage: {…}}
loadTimes: ƒ ()
app: (...)
runtime: (...)
webstore: (...)
get app: ƒ nativeGetter()
set app: ƒ nativeSetter()
get runtime: ƒ nativeGetter()
set runtime: ƒ nativeSetter()
get webstore: ƒ nativeGetter()
set webstore: ƒ nativeSetter()
```
因此可以进行如下形式检测。
```js
if (!window.chrome || !window.chrome.runtime) {
  // headless...
}
//修改属性即可绕过
window.navigator.chrome = {
    runtime: {},
    // etc.
  };
```
# 4. Permissions检测
无头模式下Notification.permission与navigator.permissions.query会返回相反的值。
```js
(async () => {
  const permissionStatus = await navigator.permissions.query({ name: 'notifications' });
  if(Notification.permission === 'denied' && permissionStatus.state === 'prompt') {
    // headless
  }
})();
//绕过方式如下
await page.evaluateOnNewDocument(() => {
  const originalQuery = window.navigator.permissions.query;
  return window.navigator.permissions.query = (parameters) => (
    parameters.name === 'notifications' ?
      Promise.resolve({ state: Notification.permission }) :
      originalQuery(parameters)
  );
});
```
# 5. Plugins长度检测
无头模式下navigator.plugins.length返回0。
```js
if (navigator.plugins.length === 0) {
  // headless
}
//绕过方式如下
Object.defineProperty(navigator, 'plugins', {
    get: () => [1, 2, 3, 4, 5],
  });
```
# 6. The Languages检测
navigator.languages检测方法
```js
if (!navigator.languages || navigator.languages.length === 0) {
  // headless
}
//绕过方式如下
Object.defineProperty(navigator, 'languages', {
    get: () => ['en-US', 'en'],
  });
```


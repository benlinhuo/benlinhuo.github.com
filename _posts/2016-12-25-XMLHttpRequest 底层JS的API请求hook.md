---
layout:     post
title:      XMLHttpRequest 底层JS的API请求hook
category: javascript
tags: [javascript]
description: 如果我们想要在每一个 API 请求的时候做一些事情，如日志统计，统一添加头信息等，而且对原生JS的封装使用到了多个库（如jQuery、vue-resource等），这种情况下，该怎么处理比较合理呢？
---

## 思路

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;针对上述描述的这种情况，因为底层使用了多个封装请求的库，我们不可能针对每个库都封装一遍我们想要的统计，所以就只能往更底层，即 `XMLHttpRequest`。每个库关于异步请求，也都是基于 `XMLHttpRequest` 的，所以我们只需要能 hook 住 `XMLHttpRequest` 中的各种方法和属性，这样在正常的操作之前，添加我们自己想要做的事情。原理同之前做的一个项目[网络流量统计](http://benlinhuo.cn/ios/2016/07/12/network-traffic.html)，iOS的 hook 利用的是 OC 的特性－runtime。


## hook 的实现

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其实实现挺简单的，我们重写原生的 `XMLHttpRequest` 。重写的方案就是遍历该对象的所有属性和方法，遍历过程中重写这些方法的实现。针对你需要 hook 的方法，则在实现它原生需要执行的内容前，先执行你想要做的事情（如日志统计等）；如果不是我们 hook 的方法，则直接执行原生执行的内容即可。该对象循环的方法有 `open  setRequestHeader   send   abort  getResponseHeader   getAllResponseHeaders   overrideMimeType  addEventListener  removeEventListener  dispatchEvent `，属性有： `UNSENT  OPENED   HEADERS_RECEIVED   LOADING  DONE
onreadystatechange  readyState  timeout  withCredentials   upload   responseURL
status   statusText   responseType  response   responseText   responseXML   onloadstart  
onprogress   onabort   onerror   onload   ontimeout  onloadend `

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们重写了 `XMLHttpRequest` 对象，当然也需要保存原来的该对象以便需要的时候可以恢复。以下代码就是 hook 的具体代码实现：

```javascript

(function(W) {
	var xmlHttpRequestBackup;
	//  我们可以将想要替换的方法，通过 `hookXMLHttpRequest` 方法的处理。将原生的对应方法加上需要额外处理的事情
	W.hookXMLHttpRequest = function(funcs) {

		// 在替换原生之前，先备份。以便恢复
		xmlHttpRequestBackup = xmlHttpRequestBackup || XMLHttpRequest;
		// 重写 XMLHttpRequest，原生 XMLHttpRequest 就是一个 function
		XMLHttpRequest = function () {
			this.xhr = new xmlHttpRequestBackup;
			// 遍历原生 XMLHttpRequest 的所有方法和属性
			for (var attr in this.xhr) {
				var type = typeof this.xhr[attr];
				if (type === 'function') {
					this[attr] = hookXHRFunction(attr);

				} else {
				 	// 为属性设置 get和 set 方法
					Object.defineProperty(this, attr, {
						get: getter(attr),
						set: setter(attr)
					});
				}
			}
		}

		function hookXHRFunction(func) {
			return function() {
				var arrArgs = Array.prototype.slice.call(arguments);// 对象转数组
				// 指定额外需要处理的方法
				if (funcs[func]) {
					funcs[func].call(this, arrArgs, this.xhr);
				}
				// 原生本应处理的内容
				this.xhr[func].apply(this.xhr, arrArgs);
			}
		}

		function getter(attr) {
			return function() {
				return this.xhr[attr];
			}
		}

		function setter(attr) {
			return function(fn) {
				var xhr = this.xhr;
				var self = this;
				// 指定要处理的方法集合：funcs
				if (funcs[attr]) {
					xhr[attr] = function() {
						// 非原生额外做的事情
						funcs[attr](self);
						// 原生属性做的事情
						fn.call(xhr, xhr);
					}

				} else {
					xhr[attr] = fn;
				}
			}
		}

	};

	W.unHookXMLHttpRequest = function() {
		if (xmlHttpRequestBackup) {
			XMLHttpRequest = xmlHttpRequestBackup;
		}
		xmlHttpRequestBackup = undefined; // 重置
	}

})(window);

``` 

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;做统计等额外事情的代码（也是对上述定义的 `window.hookXMLHttpRequest` 的应用）：

```javascript

// 在保证 hook-XMLHttpRequest.js 在该文件之前加载成功，该文件依赖于它

// 如下是用于记录日志，统一写的代码
hookXMLHttpRequest({
	// 属性，callback 执行，只传递过来了 xhr
	onload: function(xhr) {
		console.log('onload has called');
	},

	// 方法，callback 执行，传递了：第一部分参数是调用原生对应方法传递的参数，第二部分参数是 xhr
	open: function(args, xhr) {
		console.log("open called: method:%s,url:%s,async:%s",args[0],args[1],args[2],xhr)
	},

	send: function(args, xhr) {
		console.log('send has called');
	}
});
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如上是两个 js 文件，则一定保证第一个文件在第二个文件之前就加载。对整个前端框架而言，这两个 js 文件加载可以放在统一的一个头部html 中（一般我们都会有头html 和 尾html 内容，写不用页面只是中间不同而已）。



## 案例demo

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[[demo](https://github.com/benlinhuo/HookXMLHttpRequest)]。它用了 nodejs 作为服务端。我们需要在目录 `./HookXMLHttpRequest/HookXMLHttpRequest` 中执行命令 `npm start`开启服务端服务。浏览器端敲入 URL 为 `http://localhost:3000/html/demo-vue-resource.html`，打开 Console 面板，可以看到代码中指定打印的 log 内容。我分别用两种 API 库测试了：jQuery 和 vue-resource。使用 jQuery 库对应的 html 是 demo-jquery.html，使用 vue-resource 库对应的便是 demo-vue-resource.html。需要说明的一点是，vue-resource.html 中请求会在 Console 面板中有报错，查看具体报错内容，可以看出 vue-resource.js 中关于 `trim` 方法中没有判断传进来的参数是否为 undefined ，个人觉得1.0.3 版本的 vue-resource.js 该方法写法有点问题，不够健全。
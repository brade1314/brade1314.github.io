---
layout:       post
title:        "chrome 插件开发"
date:         2019-03-21 09:30:00
author:       "Brade"
header-style: text
header-mask:  0.3
catalog:      true
tags:
    - chrome extension
    - 浏览器插件
---

# 写在前面   
最近在看一门网页播放的课程，当一节课播放完毕，无法自动播放下一节。做为一个后端程序员，下意识就想用代码解决问题。首先想到了 `selenium`，然后发现只能针对特定浏览器，还需要下载驱动，比如`chrome`，需要 `chromedriver`，本人更熟悉 `java`，如果用 `java` 开发，那么还得虚拟机环境。。。如果他人使用，折腾个带虚拟机的程序，体积有点大(虽然 `java9` 以后可以定制)；然后想到 `chrome extension`，依赖少，轻量，前提是对web前端技术( `html，js，css`) 有一定了解。

# chrome extension
> `Chrome` 扩展程序是定制浏览体验的小型软件程序。它们使用户能够根据个人需求或偏好定制`Chrome`功能和行为。它们基于 `HTML`，`JavaScript` 和 `CSS` 等Web技术构建。    
>扩展必须实现一个狭义定义且易于理解的单一目的。单个扩展可以包括多个组件和一系列功能，只要所有内容都有助于实现共同目标。
>
>浏览器栏中的扩展程序图标的屏幕截图用户界面应该是最小的并且具有意图。它们的范围可以从简单的图标（如 `Google Mail Checker` 扩展程序）到覆盖整个页面。
>
>扩展文件压缩为用户下载和安装的单个 `.crx` 包。这意味着扩展不依赖于来自Web的内容，这与普通的Web应用程序不同。扩展程序通过`Chrome`开发人员信息中心发布，并发布到`Chrome`网上应用店。

  请戳 👉 [*官方文档*](https://developer.chrome.com/extensions)


## 核心文件说明
- 1、 **`manifest.json`**    
 	每个扩展都有一个名为 [*manifest.json*](https://developer.chrome.com/extensions/manifest) 的JSON格式的清单文件，它提供重要信息。
 	
        {
		  // Required 必需
		  "manifest_version": 2, // 版本，现在固定是这个
		  "name": "My Extension",
		  // 版本号，chrome根据版本号判断是否需要将插件更新至新的版本
		  "version": "versionString", 
		
		  // Recommended 推荐
		  "default_locale": "en",
		  "description": "A plain text description",
		  "icons": {...},
		
		  // Pick one (or none) 二选一
		  "browser_action": {...},
		  // 当某些特定页面打开才显示的图标
		  "page_action": {...},
		
		  // Optional 可选的
		  "action": ...,
		  "author": ...,
		  "automation": ...,
		  // 会一直常驻的后台JS或后台页面
		  "background": {
		    // Recommended
		    "persistent": false,
		    // Optional
		    "service_worker_script":
		  },
		  "chrome_settings_overrides": {...},
		  "chrome_ui_overrides": {
		    "bookmarks_ui": {
		      "remove_bookmark_shortcut": true,
		      "remove_button": true
		    }
		  },
		  "chrome_url_overrides": {...},
		  // commands API 用来添加快捷键,需要在 background page 上添加监听器绑定 handler
		  "commands": {...},
		  "content_capabilities": ...,
		  // 需要直接注入页面的JS
		  "content_scripts": [{...}],
		  "content_security_policy": "policyString",
		  "converted_from_user_script": ...,
		  "current_locale": ...,
		  "declarative_net_request": ...,
		  "devtools_page": "devtools.html",
		  "event_rules": [{...}],
		  "externally_connectable": {
		    "matches": ["*://*.example.com/*"]
		  },
		  "file_browser_handlers": [...],
		  "file_system_provider_capabilities": {
		    "configurable": true,
		    "multiple_mounts": true,
		    "source": "network"
		  },
		  // 插件主页，可以链接自己主页
		  "homepage_url": "http://path/to/homepage",
		  "import": [{"id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}],
		  "incognito": "spanning, split, or not_allowed",
		  "input_components": ...,
		  "key": "publicKey",
		  "minimum_chrome_version": "versionString",
		  "nacl_modules": [...],
		  "oauth2": ...,
		  "offline_enabled": true,
		  "omnibox": {
		    "keyword": "aString"
		  },
		  "optional_permissions": ["tabs"],
		  // Chrome40以前的插件配置页写法，跟下面一个参数二选一，同时存在只认后一个
		  "options_page": "options.html",
		  "options_ui": {
		    "chrome_style": true,
		    "page": "options.html"
		  },
		  // 操作需要申请的权限
		  "permissions": ["tabs"], 
		  "platforms": ...,
		  "requirements": {...},
		  "sandbox": [...],
		  // 插件名字简写
		  "short_name": "Short Name",
		  "signature": ...,
		  "spellcheck": ...,
		  "storage": {
		    "managed_schema": "schema.json"
		  },
		  "system_indicator": ...,
		  "tts_engine": {...},
		  // 如果不是通过 chrome web store 自动更新插件
		  "update_url": "http://path/to/updateInfo.xml",
		  "version_name": "aString",
		  // 提供插件pkg中某些资源是当前 web page 可以使用的
		  "web_accessible_resources": [...]
		}

+ 2、 **`Chrome API`**     
        所有的`Chrome API`都是以`chrome`对象开头，具体参数请看[*官方文档*](https://developer.chrome.com/apps/api_index)。
 
		bookmarks 操纵书签的API 

		browserAction 获取扩展图标、标题、文字、弹出页等
		
		commands 给扩展添加快捷键
		
		contextMenus 添加选项到右键弹出菜单
		
		cookies 控制cookies
		
		desktopCapture 捕获屏幕、个人窗口或标签内容
		
		downloads 下载控制
		
		events 事件相关API
		
		extension 获取扩展的各部分，也能与各部分交换信息
		
		extensionTypes 扩展的类型声明
		
		gcm 启用google云消息服务，收发消息
		
		history 历史记录控制
		
		i18n 多语言国际化支持
		
		idle 取得机器闲置状态
		
		management 管理扩展与应用
		
		notifications 通知控制
		
		pageAction 具体的页面下控制扩展图标、标题、文字、弹出页等相关内容
		
		permissions 获取拥有的权限
		
		power 请求系统常亮
		
		runtime 获取运行时相关信息，包括后台页、manifest等等
		
		sessions 查询或恢复浏览会话
		
		storage 存储相关
		
		tabs 与标签页交互
		
		vpnProvider 实现vpn客户端需要使用的东西
		
		webRequest 拦截、修改、阻塞请求
		
		windows 创建、修改、重排窗口


## 如何开始

### 1. 创建清单：扩展清单开始。创建一个名为 `manifest.json` 的文件。
- 通过导航到 `chrome://extensions` 来打开 `Extension Management` 页面。
	- 也可以通过单击 `Chrome` 菜单打开 `扩展管理` 页面，将鼠标悬停在 `更多工具` 上，然后选择 `扩展`。
+ 单击开发人员模式旁边的切换开关启用开发人员模式。
* 单击 `加载已解压的扩展程序` 按钮并选择扩展目录。

![](/img/chrome-extension/extensions_1.png)

### 2、添加后台脚本文件	    

通过创建名为background.js的文件并将其放在扩展目录中来介绍后台脚本。
必须在清单中注册后台脚本和许多其他重要组件。 在清单中注册后台脚本会告诉扩展程序要引用哪个文件，以及该文件的行为方式。

	 {
	    "name": "Getting Started Example",
	    "version": "1.0",
	    "description": "Build an Extension!",
	    "background": {
	      "scripts": ["background.js"],
	      "persistent": false
	    },
	    "manifest_version": 2
	  }

在 `background.js` 中根据需求加上事件监听和业务处理。
导航回扩展管理页面，然后单击 `重新加载` 链接。 通过蓝色链接背景页面可以使用新字段 `检查视图` 。
	
![](/img/chrome-extension/extensions_2.png)
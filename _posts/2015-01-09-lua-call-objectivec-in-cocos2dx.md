---
tags : [lua, objective-c, cocos2d-x]
---

# cocos2d-x中lua与objective-c的互相调用


最近用cocos2d-x做项目，需要接第三方的sdk，ios端提供的api都是objective-c的，而我们是用lua作为脚本语言，所以需要学习一下lua对objective-c的调用，以及回调。

开始的时候搜到了一个[Lua-Objective-C-Bridge](https://github.com/torus/Lua-Objective-C-Bridge)，不过我在使用的时候提示找不到全局变量objc。

我查了一下，LuaBridge.m中有如下语句表明设置了全局变量objc，并且在utils.lua中也可以访问。

	lua_setglobal(L, "objc");
	NSString *path = [[NSBundle mainBundle] pathForResource:@"utils" ofType:@"lua"];
	if (luaL_dofile(L, [path UTF8String])) {
		const char *err = lua_tostring(L, -1);
		NSLog(@"error while loading utils: %s", err);
	}

那么问题来了，**是不是必须按照上面的方式执行过的脚本才能使用objc全局变量？**求了解的朋友不吝指点。

于是我又查了一下，发现cocos2d-x 3.0之后已经集成了lua调用objective-c的方法，详见CCLuaObjcBridge.mm。此外，luaoc.lua又进一步进行了封装，调用方法为：

	local ok, ret = luaoc.callStaticMethod(className, methodName, args)

className和methodName是objective-c中的类名和方法名，args必须是table类型。两个返回值分别表示有没有执行成功和错误码。

## objective-c对lua的回调
如果参数args的元素是lua function，则会在lua stack中retain，并且生成一个functionId以方便调用。

CCLuaObjcBridge.mm中的代码片段如下。

    case LUA_TFUNCTION:
    	int functionId = retainLuaFunction(L, -1, NULL);
    	[dict setObject:[NSNumber numberWithInt:functionId] forKey:key];

于是可以通过lua传入回调函数，在objective-c获取functionId来进行回调。

## 示例
### lua代码

	local function callbackFunc(param)
		print('callbackFunc', param)
	end

	local luaoc = require('luaoc')
	luaoc.callStaticMethod(someClassName, someMethodName, {callback = callbackFunc})

### objective-c代码
LuaBridge::getStack()返回lua栈，我们可以通过它来执行一些lua的操作。

	+ (void) someMethodName : (NSDictionary *) dict
	{
		int functionId = [[dict objectForKey:@"callback"] intValue];
	    cocos2d::LuaBridge::pushLuaFunctionById(functionId);     //回调函数入栈
		cocos2d::LuaBridge::getStack()->pushString("from objc"); //参数入栈
	    cocos2d::LuaBridge::getStack()->executeFunction(1);      //执行函数，参数个数为1
	    cocos2d::LuaBridge::releaseLuaFunctionById(functionId);  //release之前被retain的函数
	}

当然，如果不需要立即执行回调，则可以先记录需要回调的functionId，在需要的时候再执行pushLuaFunctionById等语句。
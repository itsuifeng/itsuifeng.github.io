---
layout: post
title: "Agent技术，监控其它项目"
description:
headline:
modified: 2015-06-28 16:48:00 +0800
category: java
tags: [java]
imagefeature: 
mathjax:
chart:
featured: true
---
------
Java Agent是一个非常有趣的技术，能够在main函数执行之前由JVM先行执行Agent代理的代码，这样一来。对于我们监控Java应用的性能、统计之类的处理可以编写一个通用或公用的Agent模块，来完成监控处理。

------
如何使用Java Agent技术，首先我们需要写一个类，该类包含一个函数public static void premain(String agentArguments, Instrumentation ins) {// put some code here };
JVM会在main函数调用之前首先调用该方法，完成一些特定操作。

-----
so, some code would like this
```java
package com.sanly.agent;

import java.io.PrintStream;
import java.lang.instrument.Instrumentation;

import com.sanly.tail.JConsoleTailerFrame;

public class TailAgent {
	public static void premain(String agentArguments, Instrumentation ins) {
		LogClassTransformer transformer = new LogClassTransformer();
		ins.addTransformer(transformer);
		
		final JConsoleTailerFrame console = new JConsoleTailerFrame();
		
		final IOHandler handler = new IOHandler(new IOListener() {
			
			@Override
			public void done(String msg) {
				console.appendMessage(msg);
			}
		});
		//MutiOutputStream mos = new MutiOutputStream(System.err, handler.getStream());
		
		System.setErr(new PrintStream(handler.getStream()));
		
		Runtime.getRuntime().addShutdownHook(new Thread(){
			@Override
			public void run() {
				System.out.println("IOHandler close()...");
				handler.close();
			}
		});
	}
}
```

---
从分发挥想象中，未完待续。。。
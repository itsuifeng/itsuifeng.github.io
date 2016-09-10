---
layout: post
title: "Java利器之-多输出流MutiOutputStream"
description:
headline:
modified: 2015-06-07 14:46:18 +0800
category: java
tags: [java]
imagefeature: 
mathjax:
chart:
featured: true
---

----------

一些情况下，我希望I/O流能够一次性写入到任何输出流，比如网络、磁盘、管道、缓存、控制台、swing组件等，但是我不愿意重复写复制流的代码，于是多输出流MutiOutputStream应运而生；

----------

**应用场景:** 首先我们看一下多输出流的应用场景，我在公司工作时遇到这样一个现实场景，底层框架catch了一个异常，这个异常通过调用`e.printStackTrace()`方法直接打印到控制台；逻辑层根本无法捕获这个异常，但是现实情况是我必须拿到这个异常，通过业务逻辑重新处理这个异常。也许看到这里很多朋友，会对此呲之以鼻，觉得这个异常是底层框架没有处理好，既然业务层需要处理这个异常，设计框架时应该将这个异常抛出，由义务层捕获并处理。不过于我而言，其实我给框架的作者进行了多次交流，但是框架的作者不希望抛出此异常，因此十分无奈，正所谓道高一尺、魔高一丈。既然无法说服别人，就要用不寻常的方式解决这个问题；

----------

**处理方案：** 方案很简单，因为是偏方所以不建议大家这样做，遇到这样的问题还是最好多沟通，尽量由底层解决。我们知道`e.printStackTrace()`方法会将错误输出流定向到System.err流中，因此我将System.err输出流定向到一个·`ByteArrayOutputStream`中，对`ByteArrayOutputStream`进行解析，解析到需要的异常，直接处理掉异常；但是由此也引发一个问题，`System.err`被定向到`ByteArrayOutputStream`后，控制台将不会再打印错误信息，因此需要寻求一种方案，将`System.err`同时定向到`ByteArrayOutputStream`和`System.err`控制台本身。以下是部分实现逻辑。

----------


```
package com.sanly.niu;

import java.io.FilterOutputStream;
import java.io.IOException;
import java.io.OutputStream;

/**
 * 一个流多个输出
 * 
 * 应用场景： 既写入文件流，又写到控制台流
 * 
 * @author sanly
 * @create date 2015年5月18日
 * 
 */
public class MutiOutputStream extends FilterOutputStream {
	
	private final OutputStream out2;

	public MutiOutputStream(OutputStream out1, OutputStream out2) {
		super(out1);
		this.out2 = out2;
	}

	@Override
	public void write(int b) throws IOException {
		super.write(b);
		out2.write(b);
	}

	@Override
	public void write(byte[] b) throws IOException {
		super.write(b);
		out2.write(b);
	}

	@Override
	public void write(byte[] b, int off, int len) throws IOException {
		super.write(b, off, len);
		out2.write(b, off, len);
	}

	@Override
	public void flush() throws IOException {
		super.flush();
		out2.flush();
	}

	@Override
	public void close() throws IOException {
		super.close();
		out2.close();
	}
	
}

```
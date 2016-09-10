---
layout: post
title: "native加载"
description:
headline:
modified: 2015-06-07 13:22:06 +0800
category: java
tags: [java]
imagefeature: 
mathjax:
chart:
featured: true
------
Java项目中有时候会涉及到一些第三方native调用，加载动态链接库(.dll、.so)文件经常会遇到一些错误，如无法加载，如何解决这个问题呢？

----------
1. 通过JVM启动参数设定`java -Djava.library.path=c：/naitve`
2. 在代码中设定`System.setProperty("java.library.path", "c:/native");`(<font color="red">错误</font>);
3. 利用反射机制注入库路劲，调用编写好的类`ClassPathTool.addToClassPath("c:/native")`实现。

----------

- 第一种方式是可以正确设置库文件加载路径的，程序能够正常运行；
- 第二种方式是不行的，java.library.path只有在JVM启动时候读取一次，因此使用第二种方式无法生效；
- 第三种方式通过反射机制注入库路径，能按照我的想法正常运行，以下是如何实现。

```

public class ClassPathTool {
	
	public static void addToClassPath(File classPathFile) {
		try {
			Field field = ClassLoader.class.getDeclaredField("usr_paths");
			field.setAccessible(true);
			
			Object obj = field.get(null);
			
			List<String> list = new ArrayList<String>(Arrays.asList(((String[]) obj)));
			list.add(classPathFile.getAbsolutePath());
			for(File f : classPathFile.listFiles()) {
				list.add(f.getAbsolutePath());
			}
			String[] arr = new String[0];
			field.set(null, (String[]) list.toArray(arr));
		} catch (NoSuchFieldException e) {
			e.printStackTrace();
		} catch (SecurityException e) {
		} catch (IllegalArgumentException e) {
			e.printStackTrace();
		} catch (IllegalAccessException e) {
			e.printStackTrace();
		}
	}
}

```

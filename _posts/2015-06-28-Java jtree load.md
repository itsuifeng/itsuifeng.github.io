---
layout: post
title: "Java树型组件JTree的异步加载"
description:
headline:
modified: 2015-06-28 15:27:41 +0800
category: java
tags: [java]
imagefeature: 
mathjax:
chart:
featured: true
---
----
Java Swing编程中JTree树型组件的应用十分广泛，但是就目前JTree自身的模型是同步加载的，如何让JTree具有异步加载的能力，在JTree加载数据时，出现等待加载中的提示，且不阻塞EDT事件分派线程，使界面流畅的运行，用户的体验会得到提升。一般而言往树上加载的数据都是从数据库、文件、网络等获取，不排除长时间不响应的情况，异步加载树提升用户体验是必要的。

---
最近公司的一个项目使用JTree树型组件展示一些数据库查询出的数据，但是数据库查询优化到极致（目前认为），也需要1~3秒，这样一来，加载数据时EDT一直阻塞，导致界面出现卡死的现象，于是决定将JTree搞成异步加载的方式，长时间的数据库查询放到SwingWorker线程中，以下是部分实现。

------
异步加载节点

```java
package com.sanly.tree;

import java.util.ArrayList;
import java.util.List;

import javax.swing.SwingWorker;
import javax.swing.tree.DefaultMutableTreeNode;
import javax.swing.tree.DefaultTreeModel;
import javax.swing.tree.MutableTreeNode;

/**
 * 异步加载树节点
 * 
 * 思想： 通过给节点提供一个ChildNodeLoader子节点加载器，则加点可以自己加载子节点，切一旦加载子节点，下次不用再加载，
 * 除非调用setLoaded(false)，强制节点刷新重新加载子节点，子类继承该类，即可实现异步加载
 * 
 * @author sanly
 * @create date : 2015年6月23日
 * 
 */
@SuppressWarnings("serial")
public class AsyncTreeNode extends DefaultMutableTreeNode {
	
	/*** 是否已加载 */
	private boolean loaded = false;
	
	public AsyncTreeNode(Object userObject, boolean expansible) {
		super(userObject, expansible);
		if(expansible) {
			add(new LoadingTreeNode());
		}
	}

	/**
	 * 设置子节点
	 * 
	 * @param children
	 */
	private void setChildren(List<AsyncTreeNode> children) {
		removeAllChildren();
		if(children==null) {
			return ;
		}
		setAllowsChildren(children.size() > 0);
		for (MutableTreeNode node : children) {
			add(node);
		}
		loaded = true;
	}

	@Override
	public boolean isLeaf() {
		return !getAllowsChildren() && getChildCount() == 0;
	}

	/**
	 * 加载子节点
	 * 
	 * @param model JTree模型
	 * @param progressListener 回调接口，加载进度
	 */
	public void loadChildren(final DefaultTreeModel model, final ChildNodeLoader loader) {
		if (loaded) {
			return;
		}
		
		// 使用SwingWorker后台线程异步加载
		SwingWorker<List<AsyncTreeNode>, Void> worker = new SwingWorker<List<AsyncTreeNode>, Void>() {
			
			@Override
			protected List<AsyncTreeNode> doInBackground() throws Exception {
				// 这里查询数据库子节点
				if(loader!=null) {
					return loader.load(AsyncTreeNode.this);
				}
				
				return new ArrayList<AsyncTreeNode>(0);
			}

			@Override
			protected void done() {
				try {
					setChildren(get());
					// 通知节点数据发生变化
					model.nodeStructureChanged(AsyncTreeNode.this);
				} catch (Exception e) {
					e.printStackTrace();
					// 通知用户查询数据发生错误
				}
				super.done();
			}
		};
		worker.execute();
	}

	public boolean isLoaded() {
		return loaded;
	}

	/**
	 * 强制重新加载子节点
	 * 
	 * @param loaded
	 */
	public void setLoaded(boolean loaded) {
		this.loaded = loaded;
	}

}

```
等待加载中节点：
```java
package com.sanly.tree;

import java.io.File;

import javax.swing.ImageIcon;
import javax.swing.tree.DefaultMutableTreeNode;

/**
 * GIF旋转动画表示的等待加载中节点
 * 
 * @author sanly
 * @create date : 2015年6月28日
 * 
 */
public class LoadingTreeNode extends DefaultMutableTreeNode {
	
	private static final long serialVersionUID = 7859393369201957240L;
	
	private ImageIcon loadingIcon = new ImageIcon(new File("img/loading.gif").getAbsolutePath());
	
	public LoadingTreeNode() {
		super("加载中。。。", false);
	}

	public ImageIcon getLoadingIcon() {
		return loadingIcon;
	}

	public void setLoadingIcon(ImageIcon loadingIcon) {
		this.loadingIcon = loadingIcon;
	}

}

````
节点加载器：
```java
package com.sanly.tree;

import java.util.List;

/**
 * 节点加载器
 * 
 * @author sanly
 * @create date : 2015年6月28日
 * 
 */
public interface ChildNodeLoader {
	
	/**
	 * 通过父节点，加载子节点，返回加载后的子节点
	 * 
	 * @param parentNode 父节点
	 * @return 子节点集合
	 */
	public List<AsyncTreeNode> load(AsyncTreeNode parentNode);
	
	/**
	 * 节点加载完毕后自动回调该函数，用户可以在此函数中完成一些其它操作比如（选择节点等）
	 */
	public void loadDone();
}
```
等待旋转节点渲染器：
```java
package com.sanly.tree;

import java.awt.Component;
import java.awt.Image;
import java.awt.Rectangle;
import java.awt.image.ImageObserver;

import javax.swing.JTree;
import javax.swing.tree.DefaultTreeCellRenderer;
import javax.swing.tree.DefaultTreeModel;
import javax.swing.tree.TreePath;

/**
 * 思想：渲染异步加载Loading等待GIF图像，通过ImageObserver接口，使具有帧的GIF动画，自动更新显示区域组件<br>
 * 如果对数节点有特定的其它渲染，可以直接继承该类，但是必须在getTreeCellRendererComponent方法首先调用父类super().getTreeCellRendererComponent方法
 * 
 * @author sanly
 * @create date : 2015年6月24日
 * 
 */
@SuppressWarnings("serial")
public class AsyncTreeCellRenderer extends DefaultTreeCellRenderer {
	
	@Override
	public Component getTreeCellRendererComponent(final JTree tree, Object value, 
			boolean sel, boolean expanded, 
			boolean leaf, int row, boolean hasFocus) {
		super.getTreeCellRendererComponent(tree, value, sel, expanded, 
				leaf, row, hasFocus);
		if(value instanceof LoadingTreeNode) {
			final LoadingTreeNode node = (LoadingTreeNode) value;
			
			// 设置GIF图像的观察者，使观察者可以获取图像更新的通知，从而达到动画效果
			node.getLoadingIcon().setImageObserver(new ImageObserver() {
				
				@Override
				public boolean imageUpdate(Image img, int infoflags, int x, int y, int width, int height) {
					if((infoflags & (FRAMEBITS | ALLBITS)) != 0 ) {
						TreePath path = new TreePath(((DefaultTreeModel) tree.getModel()).getPathToRoot(node));
						Rectangle rect = tree.getPathBounds(path);
						if(rect!=null) {
							tree.repaint(rect);
						}
					}
					return (infoflags & (ABORT | ALLBITS)) == 0 ;
				}
			});
			this.setIcon(node.getLoadingIcon());
		}
		return this;
	}
}
```
主函数部分实现：
```java
final ChildNodeLoader loader = new ChildNodeLoader() {
			
	@Override
	public void loadDone() {
		// TODO Auto-generated method stub
		
	}
	
	@Override
	public List<AsyncTreeNode> load(AsyncTreeNode parentNode) {
		// 这里查询数据库子节点
		List<AsyncTreeNode> children = new ArrayList<AsyncTreeNode>();
		
		// 模拟数据库查询5条数据
		for (int i = 0; i < 5; i++) {
			// 模拟长时间查询0.3秒
			try {
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			children.add(new BizTreeNode("wanggj" + i +"@30san.com", "sanly"+i, "男", false));
		}
		return children;
	}
};

tree.addTreeWillExpandListener(new TreeWillExpandListener() {

	public void treeWillExpand(TreeExpansionEvent event) throws ExpandVetoException {
		TreePath path = event.getPath();
		if (path.getLastPathComponent() instanceof AsyncTreeNode) {
			AsyncTreeNode node = (AsyncTreeNode) path.getLastPathComponent();
			node.loadChildren(model, loader);
		}
	}

	public void treeWillCollapse(TreeExpansionEvent event) throws ExpandVetoException {

	}
});

tree.setCellRenderer(new AsyncTreeCellRenderer());
```
---
效果图：
![](http://itsuifeng.github.io/pic/asyncTree.png)
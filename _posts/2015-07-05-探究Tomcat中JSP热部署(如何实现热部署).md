---
layout: post
title: "探究Tomcat中JSP热部署机制"
date: 2015-07-04
categories: Tomcat
excerpt: 修改JSP文件之后为什么不需要重启服务器呢?
---

* content
{:toc}

## 1.探究Tomcat中JSP热部署机制

### 1.1 JSP热部署源码分析

一直对Tomcat中JSP热部署机制有着极大的兴趣,今天正好有时间写下这篇博文

首先简单说一下：

> * 准确的讲,JSP就是Servlet,JSP是一个标准的文本文件,在第一次访问时,每一个Web程序(WebApp,一个Context对应一个Web App)Context容器会将JSP文件"翻译"成Servlet,然后在进行调用.

查看conf/web.xml中的配置：

	<servlet>
		<servlet-name>jsp</servlet-name>
		<servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
		<init-param>
			<param-name>fork</param-name>
			<param-value>false</param-value>
		</init-param>
		<init-param>
			<param-name>xpoweredBy</param-name>
			<param-value>false</param-value>
		</init-param>
		<load-on-startup>3</load-on-startup>
	</servlet>
	
	<servlet-mapping>
		<servlet-name>jsp</servlet-name>
		<url-pattern>*.jsp</url-pattern>
	</servlet-mapping>

我们可以知道,当我们访问所有后缀为".jsp"等请求将会访问JspServlet(功能就是“翻译”JSP并对其实施访问);
具体conf/web.xml中其他配置或者参数含义详见博文[Tomcat服务器原理详解](http://www.cnblogs.com/mo-wang/p/3705147.html)

> 那我们可以知道JSP的入口则是JspServlet

> 知道入口就可以阅读JSP加载机制源码,那我们首先从网上下载Tomcat源码：[源码地址](https://github.com/apache/tomcat)

找到tomcat-trunk/java/org/apache/jasper/servlet/JspServlet类
JspServlet类继承HttpServlet类,重写service(request, response)方法,这个方法负责处理客户请求;

	public void service(HttpServletRequest request, HttpServletResponse response){
		
		.......
		//判断你请求jsp页面时有没有带?jsp_precompile查询字符串，如果带了就会重新编译
		boolean precompile = preCompile(request);
		/*判断jsp是否要进行编译处理,并通过jspUri获取JspServletWrapper对象
		 *若JspServletWrapper对象为空,则判断jspUrl对应的JSP文件是否存在
		 *／
		serviceJspFile(request,response,jspUri,precompile);
		.......
	}

> 找到serviceJspFile方法:

	private void serviceJspFile(HttpServletRequest request,
				HttpServletResponse response, String jspUri,
				boolean precompile){
		......
		JspServletWrapper wrapper = rctxt.getWrapper(jspUri);
		if(wrapper == null){
			synchronized(this){
				wrapper = rctxt.getWrapper(jspUri);
				......
				//判断jspUri对应的资源是否存在
				handleMissingResource(request, response, jspUri);
				......
			}
		}

		......
		wrapper.service(request, response, precompile);
		......
	}

> 我们继续看JspServletWrapper类中service方法：

	public void service(HttpServletRequest request,
			  HttpServletResponse response, boolean procompile){
		
		
		/*
		 * 校验逻辑(判断jsp是否可以使用)
		 */		
		......

		//Compile编译
		......
			//设置reload属性为true;即重新加载servlet
			ctxt.compile();
		......

		//加载Jsp文件对应的Servlet
		servlet = getServlet();
		......
	}	

> 继续看JspCompilationContext类中compile方法：

	public void compile(){
		//创建编译器
		createCompile();
		
		if(jspCompiler.isOutDated()){
			//JSP文件不存在则抛出异常
			if(isRemove())
				throw new FileNotFoundException(jspUri);
			......
			//进行编译
			jspCompiler.compiler();
			//设置reload属性true,表明jsp文件修改,需要重新加载
			jsw.setReload(true);
			......
		}		
	}

> 最后的核心方法Compiler类中的isOutDated(boolean checkClass)方法

	public boolean isOutDated(boolean checkClass){
		
		if (jsw != null
                		&& (ctxt.getOptions().getModificationTestInterval() > 0)) {
		     
                        /*
                         * 4秒缓存机制，其中ctxt.getOptions()得到的是EmbeddedServletOptions
		       * 类,这个类中定义有modificationTestInterval = 4
		       * 如果jsp文件修改了,需等到4秒之后才会重新编译加载;
                         */
    	  	      if (jsw.getLastModificationTest()
                  		  + (ctxt.getOptions().getModificationTestInterval() * 1000) 			  > System.currentTimeMillis()) {
                			return false;
            	       }
            	      jsw.setLastModificationTest(System.currentTimeMillis());
 	         }
		
		File targetFile;
		//拿到工作目录下的class文件对象
        		if (checkClass) {
           	 	targetFile = new File(ctxt.getClassFileName());
        		} else {
            		targetFile = new File(ctxt.getServletJavaFileName());
        		}
        		if (!targetFile.exists()) {
            		return true;
        		}
        		long targetLastModified = targetFile.lastModified();
        		if (checkClass && jsw != null) {
            		jsw.setServletClassLastModifiedTime(targetLastModified);
       		}
		
		/*
		 * 判断jsp文件的修改时间,实际处理就是将jsp文件的时间与class文件时间进行对比比较
		 */
        		Long jspRealLastModified = ctxt.getLastModified(ctxt.getJspFile());
        		if (jspRealLastModified.longValue() < 0) {
            		// Something went wrong - assume modification
            		return true;
        		}
		
		//比较JSP文件与class文件的时间戳
        		if (targetLastModified != jspRealLastModified.longValue()) {
            		if (log.isDebugEnabled()) {
                			log.debug("Compiler: outdated: " + targetFile + " 							"+ targetLastModified);
           		}
            		return true;
        		}
	}

> 最后总结：Tomcat热部署就是当Context容器检测JSP发生修改,就会重新新建一个类加载器重新加载JSP文件对应的Servlet;

Tomcat中JSP热部署机制时序图所示：

![13050408499760](http://xiaohuishu.net/static/post_image/jsphotswap.png)

最后贴一个大牛博客中的一张图[Tomcat的热部署的分析](http://www.blogjava.net/heavensay/archive/2013/12/03/389685.html)

<h4>Tomcat加载资源的概况图：</h4>

![tomcat](http://images.blogjava.net/blogjava_net/heavensay/classloader/loadprocess.jpg)

## 2.自定义实现热部署机制
	
参考:

* [探索Java热部署](http://xiaohuishu.net/2015/07/26/%E6%8E%A2%E7%B4%A2Java%E7%83%AD%E9%83%A8%E7%BD%B2/)

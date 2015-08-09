---
layout: post
title: "探究C#委托机制,对比Java实现"
date: 2015-05-29
categories: C#
excerpt: 什么是委托?对比Java中的事件
---

* content
{:toc}

## 公交车事故案例分析

对于这个问题：	

> 公交车如果在路途中发生事故,公交车管理中心肯定在第一时间内得到事故消息,并及时产生处理行为

通过详解C#委托机制并通过其实现这一问题,联想到Java中如何分析并实现这个问题;

### C#委托机制

委托是一个类,定义了方法的类型

> 具体详解委托机制请参考博文[C#委托机制](http://blog.csdn.net/yap111/article/details/2110544)
> 声明定义一个委托:

	public[protected,private] void delegate CallBack(<参数>);

> 委托使用

	//声明自定义委托类型的一个事件
	public event CallBack listener;
	
	//说明绑定一个方法<函数>到listener事件上(listener事件需是用CallBack委托声明)
	listener += new CallBack(<method(参数<声明委托时的参数>)>);

C#编译器为我们做如下工作：

	1.声明一个类CallBack：
	2.扩展自System.MulticastDelegate<extends System.MulticastDelegate>
	3.该类包含一个构造器
	4.该类包含3个方法：BeginInvoke,EndInvoke，Invoke

那什么是委托？

> * 那通俗的来讲,委托是用来执行方法<函数>的.再仔细分析一下上面的代码.当listener事件触发时,由于你将method方法绑定到该事件,所以会调用method方法,所以委托可以将方法当作另一个方法的参数来进行传递

比如：平常我们做公交车,如果公交车<每个公交车都有个唯一标识的ID>发生故障了,需要报告给公交车管理中心.

在程序中我们需要将公交车管理中心这个对象传递给公交车.公交车会告诉这个对象:说我出故障了,请及时处理！

> 这个我们一般的处理,但是这样做的一个缺点就是将公交车与公交车管理中心这两个对象紧密的联合在一起耦合度很高,现在我们如果利用委托,我们可以把这两个对象解耦出来

### 代码实现

#### C#委托机制

首先思考一个实现思路：

> * 1.首先定义两个实体类：1.Bus类 2.BusManager类
* 2.声明一个Bus处理委托:public void delegate BusHandler(Object sender,EventArgs e);
* 3.Bus类需要一个事件,当公交车发生故障是会触发这个事件：event BusHandler busFailCause;
* 4.BusManager类需要一个方法来处理公交车发生故障
* 5.需要自定义一个事件参数类型BusEventArgs,里面有Bus的编号等基本信息

实现代码,我是用Java来写的伪代码;

	//声明委托(公交车处理)
	public void delegate BusHandler(Object sender,EventArgs e);
	//新建一个公交车事件参数,接收公交车编号
	public BusEventArgs extends EventArgs{
		//busID编号
		private String busId;
	 	
		public BusEventArgs(String busId){
			this.busId = busId;
		}

		public String getBusId(){
			return this.busId;
		}
	}
	//公交车实体类
	public class Bus{
		private String busId;
	 	
		public void setBusId(String busId){
			this.busId = busId;
		}	
	 	
		public String getBusId(){
			return this.busId;
		}
	 	//声明公交车事故处理事件
	 	public event BusHandler busFailCause;
	 		//公交车产生事故方法,调用事故事件
	 		public void failCause(){
	 			if(busFailCause == null){
					System.out.println("事件未绑定.");
	 			}else{
	 				BusEventArgs eventArgs = new BusEventArgs(this.busId);
					busFailCause(this,eventArgs);
				}
	 		}
	 	}
	}
	//公交车处理中心类
	public class BusManager{
		public void BusFailHandler(Object sender,EventArgs e){
			System.out.println("编号：" + e.busId + "公交车出现故障,现准备处理")
		}
	}
	//客户端测试类
	public class Client{
		public static void main(String []args){
			Bus bus = new Bus();
			bus.setBusId("GB1212");
			BusManager manager = new BusManager();
			//绑定委托方法到公交车处理中心处理方法
			bus.busFailCause += new BusHandler(manager.BusFailHandler)；
			bus.failCause();
		}
	}

#### Java对比实现

	//函数接口,当公交车发生事故时,调用该方法
	public interface BusFailManager {
	 		
		void busFailProcess(String busId);
		
	}
		
	//公交车实体类
	public class Bus {
		
		private String busId;
		
		public String getBusId() {
			return busId;
		}
		
		public void setBusId(String busId) {
			this.busId = busId;
		}
		//公交车发生事故处理方法
		public void busFailProcess(BusFailManager manager) {
			
			manager.busFailProcess(this.busId);
		}
	}
	//客户端测试
	public class Client {
		public static void main(String[] args) {
			Bus bus = new Bus();
			
			bus.setBusId("GB13435");
		    	／*直接可以传入Lambda表达式作为BusFailManager对象 
			 * 因为BusFailManager接口中只有一个方法声明(允许存在default方法,但
			 *只能存在一个方法声明@FunctionInterface)
			*/
			bus.busFailProcess((busId) -> {
				if(busId == null){
				System.out.println("有公交车发生事故,请检查并及时处理！");
				}else{
					System.out.println("编号: " + busId + "公交车发生事故,请及时处理！");
				}
			});
		}
	}		

## 总结

> * 就事件处理来讲,Java在JDK8以前做的并不够好,尤其是令人恶心的Java Swing中的事件监听(为组件添加事件监听)
> * 虽然程序员可以通过反射的来进行抽象封装处理,但是本质依然一样;Java8在这一点上进行了改善引入了lambda表达式(Java8函数式编程);
> * 纵使这样相对C#委托机制来讲,依旧显得不够灵活.在上述例子中：利用委托可以将Bus类与BusManager类完全解耦出来;

---
	
> 但是在Java8中却需依赖函数接口BusFailManager,去掉了实际上的BusManager.我也曾试过将BusManager用上,但惭愧的是效果既然更差(也可能是没想到真正的好方法)!
  
可能是自身理解能力还不足,有待于日后发现如何利用java来真正的实现C#中的委托机制！Mark;

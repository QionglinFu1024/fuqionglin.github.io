---
layout: post
title: 'LLDB(Low Lever Debug)'
subtitle: '断点、命令'
date: 2018-03-09
categories: 技术
cover: 
tags: 逆向tz
---

### 断点
快捷键 `com+\`

* 设置断点
$breakpoint set -n XXX名称
set 是子命令
-n 是选项 是--name 的缩写!
	- OC单个方法设置 breakpoint set -n "-[ViewController test1]"
	- OC多个个方法设置 breakpoint set -n "-[ViewController test1]" -n "-[ViewController test2]"
	- Breakpoint 1: 2 locations.1代表第一组断点，2locations代表有2个地方，如何查看？
* 查看断点列表

<pre><code class="language-objectivec">$ breakpoint list
Current breakpoints://当前断点信息
1: names = {'-[ViewController test1]', '-[ViewController test2]'}, locations = 2, resolved = 2, hit count = 0//第一组
  1.1: where = LLDB`-[ViewController test1] + 20 at ViewController.m:24, address = 0x000000010450e690, resolved, hit count = 0 
  1.2: where = LLDB`-[ViewController test2] + 20 at ViewController.m:29, address = 0x000000010450e6bc, resolved, hit count = 0 
</code></pre>

* 禁用/启用

<pre><code class="language-objectivec">$ breakpoint disable 禁用<简写：break dis 1>
$ breakpoint enable  启用<简写：break en 1>
$ breakpoint disable 1 //禁用第一组
1 breakpoints disable.
$ breakpoint enable 1 //启用
1 breakpoints enabled.

$ breakpoint disable 1.1 //禁用第一组第一个
1 breakpoints disabled.
$ breakpoint list  //查看状态
Current breakpoints:
1: names = {'-[ViewController test1]', '-[ViewController test2]'}, locations = 2, resolved = 1, hit count = 0
  1.1: where = LLDB`-[ViewController test1] + 20 at ViewController.m:24, address = 0x000000010450e690, unresolved, hit count = 0  Options: disabled 
  1.2: where = LLDB`-[ViewController test2] + 20 at ViewController.m:29, address = 0x000000010450e6bc, resolved, hit count = 0 
</code></pre>


* 删除

<pre><code class="language-objectivec">$ breakpoint delete 组号
$breakpoint list
Current breakpoints:
1: names = {'-[ViewController test1]', '-[ViewController test2]'}, locations = 2, resolved = 1, hit count = 0
  1.1: where = LLDB`-[ViewController test1] + 20 at ViewController.m:24, address = 0x000000010450e690, unresolved, hit count = 0  Options: disabled 
  1.2: where = LLDB`-[ViewController test2] + 20 at ViewController.m:29, address = 0x000000010450e6bc, resolved, hit count = 0 

2: file = '/Users/qionglinfu/Desktop/LLDB/LLDB/ViewController.m', line = 38, exact_match = 0, locations = 1

  2.1: where = LLDB`-[ViewController touchesBegan:withEvent:] + 76 at ViewController.m:39, address = 0x000000010450e770, unresolved, hit count = 0 

$ breakpoint delete 2.1//不能删除某一组的某一个。不会被删除，只是被禁用
0 breakpoints deleted; 1 breakpoint locations disabled.


$ breakpoint delete 2 //删除第二组
1 breakpoints deleted; 0 breakpoint locations disabled.

$ breakpoint delete //删除所有断点
About to delete all breakpoints, do you want to do that?: [Y/n] y
All breakpoints removed. (2 breakpoints)
</code></pre>


通过help指令可以查看所有lldb指令

<pre><code class="language-objectivec">$ help
$ help breakpoints 子指令
</code></pre>


* 遍历整个所有项目中有这个方法名的地方

<pre><code class="language-objectivec">$ breakpoint set --selector 方法名
$ breakpoint set --selector touchesBegan:withEvent:
Breakpoint 3: 74 locations.
//有这个方法的地方都打上了断点
</code></pre>

* 设置指定文件断点

<pre><code class="language-objectivec">$ breakpoint set --file ViewController.m --selector touchesBegan:withEvent:
Breakpoint 4: where = LLDB`-[ViewController touchesBegan:withEvent:] + 76 at ViewController.m:39, address = 0x000000010450e770
</code></pre>

* 遍历整个项目中满足Game:这个字符的所有方法

<pre><code class="language-objectivec">$ breakpoint set -r Game:
</code></pre>

* LLDB执行代码

<pre><code class="language-objectivec">$ expression code <简写：p code>
$ expression self.view.subviews
(__NSArray0 *) $0 = 0x00006000000161e0 @"0 elements"
$ p self.view.backgroundColor = [UIColor redColor]
(UICachedDeviceRGBColor *) $1 = 0x0000600000465080
</code></pre>

<pre><code class="language-objectivec">$ b -n "-[ViewController touchesBegan:withEvent:]"
Breakpoint 1: where = 001--LLDB`-[ViewController touchesBegan:withEvent:] + 70 at ViewController.m:85, address = 0x0000000102e9b3c6
$ po self.models
<__NSArrayM 0x604000058f00>(
<Person: 0x604000423140>,
<Person: 0x604000423200>,
<Person: 0x604000423300>
)

$ p [self.models addObject:[[Person alloc] init]];//加数据
$ po self.models//查看
<__NSArrayM 0x604000058f00>(
<Person: 0x604000423140>,
<Person: 0x604000423200>,
<Person: 0x604000423300>,
<Person: 0x604000425da0>//新数据
)

$ p self.models.lastObject
(Person *) $2 = 0x0000604000425da0//取出变量，$2是此环境下的变量。有地址存在

$ p $2.name = @"xiaoming";//赋值
error: property 'name' not found on object of type 'id _Nullable'//报错。$2为id类型

$ p [$2 setVaule:@"xiaoming" forKey:@"name"];
error: no known method '-setVaule:forKey:'; cast the message send to the method's return type//KVC赋值也失败

正确姿势：在取出的那一刻做事情！
$ p (Person *)self.models.lastObject
(Person *) $3 = 0x0000604000425da0
$ p $3.name = @"xiaoming";
(NSTaggedPointerString *) $4 = 0xa006412031812da8 @"xiaoming"


$ p Person *p4 = [[Person alloc] init]; p4.name = @"hank"; p4.age = 18; [self.models addObject:p4];
//编辑多行代码执行 control+return
$ po self.models
<__NSArrayM 0x604000058f00>(
<Person: 0x604000423140>,
<Person: 0x604000423200>,
<Person: 0x604000423300>,
<Person: 0x604000425da0>,
<Person: 0x604000424b60>
)

</code></pre>


* 查看堆栈信息

<pre><code class="language-objectivec">$b bearTest4://更简单设置断点姿势
Breakpoint 1: where = 001--LLDB`-[ViewController bearTest4:] + 46 at ViewController.m:54, address = 0x000000010be0d0ae
$c
Process 57429 resuming
$bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x000000010be0d0ae 001--LLDB`-[ViewController bearTest4:](self=0x00007fca2bc1edc0, _cmd="bearTest4:", str=@"熊熊") at ViewController.m:54
    frame #1: 0x000000010be0d066 001--LLDB`-[ViewController bearTest3:](self=0x00007fca2bc1edc0, _cmd="bearTest3:", str=@"熊熊") at ViewController.m:51
    frame #2: 0x000000010be0cfe6 001--LLDB`-[ViewController bearTest2:](self=0x00007fca2bc1edc0, _cmd="bearTest2:", str=@"熊熊") at ViewController.m:47
    frame #3: 0x000000010be0cf66 001--LLDB`-[ViewController bearTest1:](self=0x00007fca2bc1edc0, _cmd="bearTest1:", str=@"熊熊") at ViewController.m:43
    frame #4: 0x000000010be0d3f3 001--LLDB`-[ViewController touchesBegan:withEvent:](self=0x00007fca2bc1edc0, _cmd="touchesBegan:withEvent:", touches=1 element, event=0x000060000010de30) at ViewController.m:87
    frame #5: 0x000000010d724767 UIKit`forwardTouchMethod + 340
    frame #6: 0x000000010d724602 UIKit`-[UIResponder touchesBegan:withEvent:] + 49
    frame #7: 0x000000010d56ce1a UIKit`-[UIWindow _sendTouchesForEvent:] + 2052
    frame #8: 0x000000010d56e7c1 UIKit`-[UIWindow sendEvent:] + 4086
    frame #9: 0x000000010d512310 UIKit`-[UIApplication sendEvent:] + 352
    frame #10: 0x000000010de536af UIKit`__dispatchPreprocessedEventFromEventQueue + 2796
    frame #11: 0x000000010de562c4 UIKit`__handleEventQueueInternal + 5949
    frame #12: 0x000000010d01cbb1 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17
    frame #13: 0x000000010d0014af CoreFoundation`__CFRunLoopDoSources0 + 271
    frame #14: 0x000000010d000a6f CoreFoundation`__CFRunLoopRun + 1263
    frame #15: 0x000000010d00030b CoreFoundation`CFRunLoopRunSpecific + 635
    frame #16: 0x00000001121eea73 GraphicsServices`GSEventRunModal + 62
    frame #17: 0x000000010d4f7057 UIKit`UIApplicationMain + 159
    frame #18: 0x000000010be0d50f 001--LLDB`main(argc=1, argv=0x00007ffee3df2038) at main.m:14
    frame #19: 0x0000000110ad7955 libdyld.dylib`start + 1
    frame #20: 0x0000000110ad7955 libdyld.dylib`start + 1
    
    $up//回退，第三方只能看到汇编
frame #1: 0x000000010be0d066 001--LLDB`-[ViewController bearTest3:](self=0x00007fca2bc1edc0, _cmd="bearTest3:", str=@"熊熊") at ViewController.m:51
   48  	}
   49  	-(void)bearTest3:(NSString *)str{
   50  	    NSLog(@"%@",str);
-> 51  	    [4m[[0mself bearTest4:str];
   52  	}
   53  	-(void)bearTest4:(NSString *)str{
   54  	    NSLog(@"%@",str);

$down //下一个
frame #0: 0x000000010be0d0ae 001--LLDB`-[ViewController bearTest4:](self=0x00007fca2bc1edc0, _cmd="bearTest4:", str=@"熊熊") at ViewController.m:54
   51  	    [self bearTest4:str];
   52  	}
   53  	-(void)bearTest4:(NSString *)str{
-> 54  	    NSLog(@"%@",[4ms[0mtr);
   55  	    
   56  	}
   57  
   
  $ frame select 3 //根据frame定位
frame #3: 0x000000010be0cf66 001--LLDB`-[ViewController bearTest1:](self=0x00007fca2bc1edc0, _cmd="bearTest1:", str=@"熊熊") at ViewController.m:43
   40  	
   41  	-(void)bearTest1:(NSString *)str{
   42  	    NSLog(@"%@",str);
-> 43  	    [4m[[0mself bearTest2:str];
   44  	}
   45  	-(void)bearTest2:(NSString *)str{
   46  	    NSLog(@"%@",str)	
   
   $ frame variable//整个方法的参数，OC方法前面两个为隐式参数
(ViewController *) self = 0x00007fca2bc1edc0
(SEL) _cmd = "bearTest2:"
(__NSCFConstantString *) str = 0x000000010be0f178 @"熊熊"
	 $p str = @"2222222";//修改参数
(NSTaggedPointerString *) $0 = 0xa323232323232327 @"2222222"

$ thread return//代码回滚，直接返回，不在执行后面的代码<up、down只是查看>
</code></pre>

### 流程控制
###### 源码指令级别
* 继续执行
$ c continue 
* 单步运行,将子函数当做整体一步执行
$ n next
* 单步运行,遇到子函数会进去
$ s 

###### 汇编指令级别<control+功能键>
* si
* ni 

### 内存断点
* 对象/属性设置断点，类似KVO

<pre><code class="language-objectivec">$ watchpoint set variable p1->_name
Watchpoint created: Watchpoint 1: addr = 0x60000022a570 size = 8 state = enabled type = w
    declare @ '/Users/.../ViewController.m:64'
    watchpoint spec = 'p1->_name'
    new value: 0x000000010d25b118
    
    触发时打印：
    Watchpoint 1 hit:
old value: 0x000000010d25b118
new value: 0x000000010d25b178
//查看
$ po 0x000000010d25b118
one
$ po 0x000000010d25b178
hello
//调用堆栈
bt
* thread #1, queue = 'com.apple.main-thread', stop reason = watchpoint 1
  * frame #0: 0x000000010db6b4aa libobjc.A.dylib`objc_setProperty_nonatomic_copy + 47
    frame #1: 0x000000010d259597 001--LLDB`-[Person setName:](self=0x000060000022a560, _cmd="setName:", name=@"hello") at Person.h:12
    frame #2: 0x000000010d2593dc 001--LLDB`-[ViewController touchesBegan:withEvent:](self=0x00007fa4b3419080, _cmd="touchesBegan:withEvent:", touches=1 element, event=0x0000604000108040) at ViewController.m:87
    frame #3: 0x000000010eb72767 UIKit`forwardTouchMethod + 340
    frame #4: 0x000000010eb72602 UIKit`-[UIResponder touchesBegan:withEvent:] + 49
    frame #5: 0x000000010e9bae1a UIKit`-[UIWindow _sendTouchesForEvent:] + 2052
    frame #6: 0x000000010e9bc7c1 UIKit`-[UIWindow sendEvent:] + 4086
    frame #7: 0x000000010e960310 UIKit`-[UIApplication sendEvent:] + 352
    frame #8: 0x000000010f2a16af UIKit`__dispatchPreprocessedEventFromEventQueue + 2796
    frame #9: 0x000000010f2a42c4 UIKit`__handleEventQueueInternal + 5949
    frame #10: 0x000000010e46abb1 CoreFoundation`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17
    frame #11: 0x000000010e44f4af CoreFoundation`__CFRunLoopDoSources0 + 271
    frame #12: 0x000000010e44ea6f CoreFoundation`__CFRunLoopRun + 1263
    frame #13: 0x000000010e44e30b CoreFoundation`CFRunLoopRunSpecific + 635
    frame #14: 0x000000011363ca73 GraphicsServices`GSEventRunModal + 62
    frame #15: 0x000000010e945057 UIKit`UIApplicationMain + 159
    frame #16: 0x000000010d25950f 001--LLDB`main(argc=1, argv=0x00007ffee29a6038) at main.m:14
    frame #17: 0x0000000111f25955 libdyld.dylib`start + 1
    frame #18: 0x0000000111f25955 libdyld.dylib`start + 1

</code></pre>

* 通过内存地址设置

<pre><code class="language-objectivec">$ frame variable 
(ViewController *) self = 0x00007fdfd5d16020
(SEL) _cmd = "viewDidLoad"
(Person *) p1 = 0x000060400003dd00
(Person *) p2 = 0x4089600000000000
(Person *) p3 = nil
$ p &p1->_name
(NSString **) $0 = 0x000060400003dd10
$ watchpoint set expression 0x000060400003dd10
Watchpoint created: Watchpoint 1: addr = 0x60400003dd10 size = 8 state = enabled type = w
    new value: 4482445592
$ watchpoint list
Number of supported hardware watchpoints: 4
Current watchpoints:
Watchpoint 1: addr = 0x60400003dd10 size = 8 state = enabled type = w
    new value: 4482445592
$ watchpoint delete
About to delete all watchpoints, do you want to do that?: [Y/n] y
All watchpoints removed. (1 watchpoints)
</code></pre>

* 给断点设置默认指令

<pre><code class="language-objectivec">$ break command add 2//2为断点编号
Enter your debugger command(s).  Type 'DONE' to end.
> po self
> p self.view
> DONE

$ break command list 2//查看
Breakpoint 2:
    Breakpoint commands:
      po self
      p self.view
      
$ breakpoint command delete 2//删除
$ break command list 2 
Breakpoint 2 does not have an associated command.
</code></pre>
### stop-hook
让你在每次stop的时候去执行一些命令,只对breadpoint,watchpoint



### 常用命令
* image list 
* p expression的简写 执行代码
* b -[xxx xxx]
* x 
* register read
* po


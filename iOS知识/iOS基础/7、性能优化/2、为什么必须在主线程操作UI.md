# 为什么必须在主线程操作UI

在开发过程中，我们或多或少会不经意在后台线程中调用了UIKit框架的内容，可能是在网络回调时直接`imageView.image = anImage`，也有可能是不小心在后台线程中调用了`UIApplication.sharedApplication`。而这个时候编译器会报出一个runtime错误，我们也会迅速的对其进行修正。

但仔细去思考，究竟为什么一定要在主线程操作UI呢？如果在后台线程对UI进行操作会发生什么？在后台线程对UI进行操作不是可以更好的避免卡顿吗？这篇文章就是基于这样一些疑问而产生的。

> **太长不看版：**
>
> UIKit并不是一个 **线程安全** 的类，UI操作涉及到渲染访问各种View对象的属性，如果异步操作下会存在读写问题，而为其加锁则会耗费大量资源并拖慢运行速度。另一方面因为整个程序的起点`UIApplication`是在主线程进行初始化，所有的用户事件都是在主线程上进行传递（如点击、拖动），所以view只能在主线程上才能对事件进行响应。而在渲染方面由于图像的渲染需要以60帧的刷新率在屏幕上 **同时** 更新，在非主线程异步化的情况下无法确定这个处理过程能够实现同步更新。

## 从UIKit线程不安全说起

在UIKit中，很多类中大部分的属性都被修饰为`nonatomic`，这意味着它们不能在多线程的环境下工作，而对于UIKit这样一个庞大的框架，将其所有属性都设计为线程安全是不现实的，这可不仅仅是简单的将`nonatomic`改成`atomic`或者是加锁解锁的操作，还涉及到很多的方面：

- 假设能够异步设置view的属性，那我们究竟是希望这些改动能够同时生效，还是按照各自runloop的进度去改变这个view的属性呢？
- 假设`UITableView`在其他线程去移除了一个cell，而在另一个线程却对这个cell所在的index进行一些操作，这时候可能就会引发crash。
- 如果在后台线程移除了一个view，这个时候runloop周期还没有完结，用户在主线程点击了这个“将要”消失的view，那么究竟该不该响应事件？在哪条线程进行响应？

仔细思考，似乎能够多线程处理UI并没有给我们开发带来更多的便利，假如你代入了这些情景进行思考，你很容易得出一个结论： **“我在一个串行队列对这些事件进行处理就可以了。”** 苹果也是这样想的，所以UIKit的所有操作都要放到主线程串行执行。

在[Thread-Safe Class Design](https://link.juejin.cn/?target=https%3A%2F%2Fwww.objc.io%2Fissues%2F2-concurrency%2Fthread-safe-class-design%2F)一文提到：

> It’s a conscious design decision from Apple’s side to not have UIKit be thread-safe. Making it thread-safe **wouldn’t buy you much in terms of performance;** it would in fact make many things slower. And the fact that UIKit is tied to the main thread makes it very easy to write concurrent programs and use UIKit. All you have to do is make sure that calls into UIKit are always made on the main thread.

大意为把UIKit设计成线程安全并不会带来太多的便利，也不会提升太多的性能表现，甚至会因为加锁解锁而耗费大量的时间。事实上并发编程也没有因为UIKit是线程不安全而变得困难，我们所需要做的只是要确保UI操作在主线程进行就可以了。

> 好吧，那假设我们用黑魔法祝福了UIKit，这个UIKit能够完美的解决我们上面提到的问题，并能够按照开发者的想法随意展现不同的形态。那这个时候我们可以在后台线程操作UI了嘛？

**很可惜，还是不行。**

## Runloop 与绘图循环

道理我们都懂，那这个究竟跟我们不能在后台线程操作UI有什么关系呢？

`UIApplication`在主线程所初始化的Runloop我们称为`Main Runloop`，它负责处理app存活期间的大部分事件，如用户交互等，它一直处于不断处理事件和休眠的循环之中，以确保能尽快的将用户事件传递给GPU进行渲染，使用户行为能够得到响应，画面之所以能够得到不断刷新也是因为`Main Runloop`在驱动着。

而每一个view的变化的修改并不是立刻变化，相反的会在当前run loop的结束的时候统一进行重绘，这样设计的目的是为了能够在一个runloop里面处理好所有需要变化的view，包括resize、hide、reposition等等，所有view的改变都能在同一时间生效，这样能够更高效的处理绘制，这个机制被称为**绘图循环（View Drawing Cycle)**。

假设这个时候我们应用了我们的魔法UIKit，并愉快的在一条后台线程操作UI，但当我们需要对设备进行旋转并重新布局的时候，问题来了，因为各个线程之间不同步，这时候各个view修改的请求时机是零碎的，所以所有的旋转变化并不能在`Main Runloop`的一个runloop里面处理完，这就导致设备旋转之后还有一些view迟迟没有旋转。

另一方面，因为我们的魔法UIKit并不是在主线程，所以`Main Runloop`中的事件需要跨线程进行传输，这样会导致显示与用户事件并不同步。试想一下我们用我们的魔法UIKit写了一个游戏，用户如果在图片还没有加载出来的时候按下了按钮，他们就能胜利，于是我们写出了这样的代码：

**game.m**

```objc
- (void)didClickButton:(UIButton *)button
{
	if (self.imageView.image != nil) {
		// User lose!
	} else {
		// User Win!
	}
}

- (void)loadImageInBackgroundThread
{
	dispatch_async(dispatch_queue_create("BackgroundQueue", NULL), ^{
		self.imageView.image = [self downloadedImage];
	};
}

复制代码
```

因为我们完美的魔法UIKit，在后台执行`imageView.image = xxx`并不会产生任何问题。游戏上线，在你还为后台处理UI而沾沾自喜的时候，用户投诉了他们明明没有看到图片显示，点击的时候还是告诉他们输了，于是你的产品就这样扑街了。

这是因为点击等事件是由系统传递给`UIApplication`中，并在`Main Runloop`中进行处理与响应，但是由于UI在后台线程中进行处理，所以他跟事件响应并不同步。即使在UI所在的后台线程也自己维护了一个Runloop，在Runloop结束时候进行渲染，但可能用户已经进行了点击操作并开始辱骂你的游戏了。



> 好吧，那假设我天赋异禀，把整套UIApplication的机制全都重写了，也用黑魔法祝福了我的新UIApplication，这个时候它能完美的解决线程同步的问题，这个时候我可以在后台操作UI了吗？

**……**

**……**

**很可惜，还是不能。**

## 理解iOS的渲染流程

要回答这个问题，我们要先从最底层的渲染说起。

### 渲染系统框架

![img](https://user-gold-cdn.xitu.io/2019/1/17/1685badd6f86fdb4)

- UIKit: 包含各种控件，负责对用户操作事件的响应，本身并不提供渲染的能力
- Core Animation: 负责所有视图的绘制、显示与动画效果
- OpenGL ES: 提供2D与3D渲染服务
- Core Graphics: 提供2D渲染服务
- Graphics Hardware: 指GPU

所以在iOS中，所有视图的现实与动画本质上是由 **Core Animation** 负责，而不是UIKit。

### Core Animation Pipeline 流水线

![img](https://user-gold-cdn.xitu.io/2019/1/17/1685badf08f1dda1)

Core Animation的绘制是通过Core Animation Pipeline实现，它以流水线的形式进行渲染，具体分为四个步骤：

- Commit Transaction:

  可以细分为

  - Layout: 构建视图布局如`addSubview`等操作
  - Display: 重载`drawRect:`进行时图绘制，该步骤使用CPU与内存
  - Prepare: 主要处理图像的解码与格式转换等操作
  - Commit: 将Layer递归打包并发送到Render Server

- Render Server:

  负责渲染工作，会解析上一步Commit Transaction中提交的信息并反序列化成渲染树（render tree)，随后根据layer的各种属性生成绘制指令，并在下一次VSync信号到来时调用OpenGL进行渲染。

- GPU:

  GPU会等待显示器的VSync信号发出后才进行OpenGL渲染管线，将3D几何数据转化成2D的像素图像和光栅处理，随后进行新的一帧的渲染，并将其输出到缓冲区。

- Dispaly:

  从缓冲区中取出画面，并输出到屏幕上。

**知识补充：iOS的VSync与双缓冲机制**

**VSync:**

> VSync（vertical sync）是指**垂直同步**，在玩游戏的时候在设置的时候应该会看见过这个选项，这个机制能够让显卡和显示器保持在一个相同的刷新率从而避免画面撕裂。在iOS中，屏幕具有60Hz的刷新率，这意味着它每秒需要显示60张不同的图片（帧），但GPU并没有一个确定的刷新率，在某些时候GPU可能被要求更强力的数据输出来确保渲染能力，这时候他们可能比屏幕刷新率（60Hz）更快，就会导致屏幕不能完整的渲染所有GPU给他的数据，因为它不够快，屏幕的上一帧还没渲染完，下一帧就已经到来了，这就导致画面的撕裂。
>
> 这个时候我们就要引入VSync了，简单来说它就是让显卡保持他的输出速率不高于屏幕的刷新率，启用了VSync后，GPU不再会给你可怜的60Hz屏幕每秒发送100帧了，它会增加每一帧的发送间隔，确保显示器能够有充足的时间去处理每一帧。

**双缓冲机制：**

> 双缓冲机制是用于避免或减少画面闪烁的问题，在单缓冲的情况下，GPU输出了一帧画面，缓冲区就需要马上获取这个画面，并交给显示屏去显示，而这段时间GPU输出的画面就全都丢失了，因为没有缓冲区去承载这些画面，就会造成画面的闪烁。
>
> 而在双缓冲机制下有一个`Back Frame Buffer`和一个`Front Frame Buffer`，在GPU绘制完成后，它会将图像先保存到`Back Frame Buffer`中，操作完毕后，会调用一个交换函数，让绘制完成的`Back Frame Buffer`上的图像交换到`Front Frame Buffer`上。由于双缓冲利用了更多显存与CPU消耗时间，从而避免了画面的闪烁。

### So？

相信大家都会遇到过应用卡顿，卡顿的原因就是因为两帧的刷新时间间隔大于60帧每秒（约16.67ms），导致用户感觉点击或者滑动时，界面没有及时的响应。

前面提到**Core Animation Pipeline**是以**流水线**的形式工作的，在理想的状况下我们希望它能够在1/60s内完成图层树的准备工作并提交给渲染进程，而渲染进程在下一次VSync信号到来的时候提交给GPU进行渲染，并在1/60s内完成渲染，这样就不会产生任何的卡顿。

但是由于我们使用了我们的魔法UIKit，所以我们在许多后台线程进行了UI操作，在runloop的结尾准备进行渲染的时候，不同线程提交了不同的渲染信息，于是我们就拥有了更多的绘制事务，这个时候**Core Animation Pipeline**会不断将信息提交，让GPU进行渲染，由于绘制事件的不同步导致了GPU渲染的不同步，可能在上一帧是需要渲染一个label消失的画面，下一帧却又需要渲染这个label改变了文字，最终导致的是界面的不同步。

（如果你真的想要这样的效果，可以尝试一下使用我的[DWAnimatedLabel](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FDywane%2FDWAnimatedLabel)）

另一方面，在VSync和双缓冲机制我们可以看出渲染其实是一个十分消耗系统资源的操作（占用显存与CPU），所以可能会因为大量的事务和线程之间频繁的上下文切换导致了GPU无法处理，反而影响了性能，从而导致在1/60s中无法完成图层树的提交，导致了严重的卡顿。

> 但我真的很想在后台线程操作UI，我能再用黑魔法吗？

**好吧，其实是有办法的。**

## Texture or ComponentKit

[AsyncDisplayKit（现命名为Texture）](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Ffacebookarchive%2FAsyncDisplayKit) 是Facebook开源的一个用于保持iOS界面流畅的框架。

[ComponentKit](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Ffacebook%2Fcomponentkit)是Facebook开源的一个基于React思想的iOS原生UI开发框架。它通过函数式和声明的方式构建UI。

让我们撤销掉我们对UIKit施展的各种魔法，回到这个UI只能在主线程进行操作的世界吧。这两个框架其实并不是真正的在后台线程操作UI，而是用了更巧妙的方法将一些耗时的操作异步执行，从而绕开了UIKit只能在主线程操作的限制。

比如Texture创建了各类`Node`，在node中包含了UIView，而Node本身是线程安全的，所以允许在后台线程对Node进行修改，随后在第一次主线程访问View的时候它才会在内部生成对应的View，当node的属性发生改变的时候，他也不会马上进行修改，而是在适当的时机一次性的在主线程为内部的View进行设置。（有点类似于绘图循环）

而ComponentKit则是通过创建Component来描述UI，它也是一个线程安全的类。可以将Component认为是一个刻板，而UIView是刻板下的一张纸，渲染则是喷墨的过程。当我们生成了一个Component的时候，就等于生成了一个View的模版，在进行渲染的时候只要按照模版进行绘制就可以了。复杂的界面可以通过各种简单的Component来组成。（类似于Flutter的widget）

> 但是我……

**闭嘴吧你**

## 总结

UIKit只能在主线程进行操作，这一个铁律只要是熟悉iOS开发的都会有所耳闻，但是往深一层其实这个涉及到很多的东西，包括软件、整体UIKit框架的实现、硬件等等，很多细节的东西往往是我们在平常有所忽略的。可能我们知道不能在主线程操作，却不知道其内在原因；可能我们知道怎么排查、处理卡顿，却不知道其真正的成因；可能我们知道`drawRect:`方法会导致CPU飙升，却不知道原因是上下文的切换导致……

写代码从来都不是一件简单而显而易见的事情。


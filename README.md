# android-study
Android 显示原理简介
http://djt.qq.com/article/view/987
现在越来越多的应用开始重视流畅度方面的测试，了解Android应用程序是如何在屏幕上显示的则是基础中的基础，就让我们一起看看小小屏幕中大大的学问。这也是我下篇文章——《Android应用流畅度测试分析》的基础。

 

    首先，用一句话来概括一下Android应用程序显示的过程：Android应用程序调用SurfaceFlinger服务把经过测量、布局和绘制后的Surface渲染到显示屏幕上。

 

名词解释

SurfaceFlinger：Android系统服务，负责管理Android系统的帧缓冲区，即显示屏幕。

Surface：Android应用的每个窗口对应一个画布（Canvas），即Surface，可以理解为Android应用程序的一个窗口。

 

    Android应用程序的显示过程包含了两个部分（应用侧绘制、系统侧渲染）、两个机制（进程间通讯机制、显示刷新机制），接下来我们就来一一道来。

 

应用侧

一个Android应用程序窗口里面包含了很多UI元素，这些UI元素是以树形结构来组织的，即它们存在着父子关系，其中，子UI元素位于父UI元素里面，如下图：

 

 

因此，在绘制一个Android应用程序窗口的UI之前，我们首先要确定它里面的各个子UI元素在父UI元素里面的大小以及位置。确定各个子UI元素在父UI元素里面的大小以及位置的过程又称为测量过程和布局过程。因此，Android应用程序窗口的UI渲染过程可以分为测量、布局和绘制三个阶段。如下图所示：

测量：递归（深度优先）确定所有视图的大小（高、宽）

    布局：递归（深度优先）确定所有视图的位置（左上角坐标）

    绘制：在画布canvas上绘制应用程序窗口所有的视图

 

测量、布局没有太多要说的，这里要着重说一下绘制。Android目前有两种绘制模型：基于软件的绘制模型和硬件加速的绘制模型（从Android 3.0开始全面支持）。

 

    在基于软件的绘制模型下，CPU主导绘图，视图按照两个步骤绘制：

1.      让View层次结构失效

2.      绘制View层次结构

    当应用程序需要更新它的部分UI时，都会调用内容发生改变的View对象的invalidate()方法。无效（invalidation）消息请求会在View对象层次结构中传递，以便计算出需要重绘的屏幕区域（脏区）。然后，Android系统会在View层次结构中绘制所有的跟脏区相交的区域。不幸的是，这种方法有两个缺点：

1.      绘制了不需要重绘的视图（与脏区域相交的区域）

2.      掩盖了一些应用的bug（由于会重绘与脏区域相交的区域）

    注意：在View对象的属性发生变化时，如背景色或TextView对象中的文本等，Android系统会自动的调用该View对象的invalidate()方法。

 

    在基于硬件加速的绘制模式下，GPU主导绘图，绘制按照三个步骤绘制：

1.      让View层次结构失效

2.      记录、更新显示列表

3.      绘制显示列表

这种模式下，Android系统依然会使用invalidate()方法和draw()方法来请求屏幕更新和展现View对象。但Android系统并不是立即执行绘制命令，而是首先把这些View的绘制函数作为绘制指令记录一个显示列表中，然后再读取显示列表中的绘制指令调用OpenGL相关函数完成实际绘制。另一个优化是，Android系统只需要针对由invalidate()方法调用所标记的View对象的脏区进行记录和更新显示列表。没有失效的View对象则能重放先前显示列表记录的绘制指令来进行简单的重绘工作。

使用显示列表的目的是，把视图的各种绘制函数翻译成绘制指令保存起来，对于没有发生改变的视图把原先保存的操作指令重新读取出来重放一次就可以了，提高了视图的显示速度。而对于需要重绘的View，则更新显示列表，以便下次重用，然后再调用OpenGL完成绘制。

硬件加速提高了Android系统显示和刷新的速度，但它也不是万能的，它有三个缺陷：

1.      兼容性（部分绘制函数不支持或不完全硬件加速，参见文章尾）

2.      内存消耗（OpenGL API调用就会占用8MB，而实际上会占用更多内存）

3.      电量消耗（GPU耗电）

 

系统侧

    Android应用程序在图形缓冲区中绘制好View层次结构后，这个图形缓冲区会被交给SurfaceFlinger服务，而SurfaceFlinger服务再使用OpenGL图形库API来将这个图形缓冲区渲染到硬件帧缓冲区中。

 

    由于Android应用程序很少能涉及到Android系统底层，所以SurfaceFlinger服务的执行过程不做过多的介绍。

 

进程间通讯机制

    Android应用程序为了能够将自己的UI绘制在系统的帧缓冲区上，它们就必须要与SurfaceFlinger服务进行通信，如图所示：

Android应用程序与SurfaceFlinger服务是运行在不同的进程中的，因此，它们采用某种进程间通信机制来进行通信。由于Android应用程序在通知SurfaceFlinger服务来绘制自己的UI的时候，需要将UI数据传递给SurfaceFlinger服务，例如，要绘制UI的区域、位置等信息。一个Android应用程序可能会有很多个窗口，而每一个窗口都有自己的UI数据，因此，Android系统的匿名共享内存机制就派上用场了。

每一个Android应用程序与SurfaceFlinger服务之间，都会通过一块匿名共享内存来传递UI数据，如下所示：

 

 

但是单纯的匿名共享内存在传递多个窗口数据时缺乏有效的管理，所以匿名共享内存就被抽象为一个更上流的数据结构SharedClient，如下图所示：

 

在每个SharedClient中，最多有31个SharedBufferStack，每个SharedBufferStack都对应一个Surface，即一个窗口。这样，我们就可以知道为什么每一个SharedClient里面包含的是一系列SharedBufferStack而不是单个SharedBufferStack：一个SharedClient对应一个Android应用程序，而一个Android应用程序可能包含有多个窗口，即Surface。从这里也可以看出，一个Android应用程序至多可以包含31个窗口。

每个SharedBufferStack中又包含了N个缓冲区（<4.1 N=2; >=4.1 N=3）,即显示刷新机制中即将提到的双缓冲和三重缓冲技术。

显示刷新机制

    一般我们在绘制UI的时候，都会采用一种称为“双缓冲”的技术。双缓冲意味着要使用两个缓冲区（SharedBufferStack中），其中一个称为Front Buffer，另外一个称为Back Buffer。UI总是先在Back Buffer中绘制，然后再和Front Buffer交换，渲染到显示设备中。理想情况下，这样一个刷新会在16ms内完成（60FPS），下图就是描述的这样一个刷新过程（Display处理前Front Buffer，CPU、GPU处理Back Buffer。

 

 

但现实情况并非这么理想。

1.      时间从0开始，进入第一个16ms：Display显示第0帧，CPU处理完第一帧后，GPU紧接其后处理继续第一帧。三者互不干扰，一切正常。

2.      时间进入第二个16ms：因为早在上一个16ms时间内，第1帧已经由CPU，GPU处理完毕。故Display可以直接显示第1帧。显示没有问题。但在本16ms期间，CPU和GPU却并未及时去绘制第2帧数据（注意前面的空白区），而是在本周期快结束时，CPU/GPU才去处理第2帧数据。

3.      时间进入第3个16ms，此时Display应该显示第2帧数据，但由于CPU和GPU还没有处理完第2帧数据，故Display只能继续显示第一帧的数据，结果使得第1帧多画了一次（对应时间段上标注了一个Jank）。

    通过上述分析可知，此处发生Jank的关键问题在于，为何第1个16ms段内，CPU/GPU没有及时处理第2帧数据？原因很简单，CPU可能是在忙别的事情，不知道该到处理UI绘制的时间了。可CPU一旦想起来要去处理第2帧数据，时间又错过了！

 

    为解决这个问题，Android 4.1中引入了VSYNC，这类似于时钟中断。结果如下图所示：

 

由上图可知，每收到VSYNC中断，CPU就开始处理各帧数据。整个过程非常完美。

    不过，仔细琢磨后却会发现一个新问题：上图中，CPU和GPU处理数据的速度似乎都能在16ms内完成，而且还有时间空余，也就是说，CPU/GPU的FPS（帧率，Frames Per Second）要高于Display的FPS。确实如此。由于CPU/GPU只在收到VSYNC时才开始数据处理，故它们的FPS被拉低到与Display的FPS相同。但这种处理并没有什么问题，因为Android设备的Display FPS一般是60，其对应的显示效果非常平滑。

    如果CPU/GPU的FPS小于Display的FPS，会是什么情况呢？请看下图：

 

由上图可知：

1.      在第二个16ms时间段，Display本应显示B帧，但却因为GPU还在处理B帧，导致A帧被重复显示。

2.      同理，在第二个16ms时间段内，CPU无所事事，因为A Buffer被Display在使用。B Buffer被GPU在使用。注意，一旦过了VSYNC时间点，CPU就不能被触发以处理绘制工作了。

    为什么CPU不能在第二个16ms处开始绘制工作呢？原因就是只有两个Buffer（Android 4.1之前）。如果有第三个Buffer的存在，CPU就能直接使用它，而不至于空闲。出于这一思路就引出了三重缓冲区（Android 4.1）。结果如下图所示：

由上图可知：

第二个16ms时间段，CPU使用C Buffer绘图。虽然还是会多显示A帧一次，但后续显示就比较顺畅了。

是不是Buffer越多越好呢？回答是否定的。由上图可知，在第二个时间段内，CPU绘制的第C帧数据要到第四个16ms才能显示，这比双Buffer情况多了16ms延迟。所以，Buffer最好还是两个，三个足矣。

到这里，Android系统的显示原理就介绍完了。那么在了解这些原理后对我们的流畅度测试有哪些帮助呢，请看我的下篇文章《Android应用流畅度测试分析》。

 

 

附：不同的API Level下，绘制函数对硬件加速模式的支持情况

 


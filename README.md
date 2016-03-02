#Android之GPU过度绘制与图形渲染优化

##写在前面的话

本文主要对过度绘制和图形渲染做一个概念性的描述，和简单的优化措施。
如果你已对过度绘制有过一些了解，那么你应该明白，仅是简单的层级优化对过度绘制的改善是很小的。所以，这时候你可以参考这篇文章：

另外如果你还想知道更多关于View优化原理，可以参考这篇文章：
http://www.oschina.net/news/60157/android-performance-patterns

##1.	概念

<br/>
**GPU过度绘制：**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;是指在一个像素点上绘制多次（超过一次）。举一个简单的例子：显示一个什么都没有做的activity界面算作画了1层，给activity加一个背景是第2层，在上面放了一个Text View（有背景的Text View）是第3层，Text View显示文本就是第4层 。
	
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**仅仅只是为了显示一个文本，却在同一个像素点绘制了四次,这是一定要优化的！**

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;还有，过度绘制对动画性能的影响是极其严重的。如果你想要流畅的动画效果，那么一定不能忽视过度绘制！！



**图形渲染优化：**		

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个View的绘制过程：测量、布局、画图。三者的累积时间，就是一个View的最终绘制时间 。 过多的层级、无用的子节点父节点、过于依赖系统计算位置的布局属性（如： weight）。都会引起上述三个过程时间的增加。


##2.关键点/字
<br/>

- 过渡绘制优化与图形渲染优化都其目的都是为了提供一个高效的UI。其目的相似，优化方式也有相同之处，所以一起进行总结。

- 调试GPU过渡绘制颜色区域说明    

		无/白色：绘制1次
	    蓝色：绘制2次（理想状态）
	    绿色：绘制3次
	    浅红：绘制4次（要优化了）
	    深红：绘制5次或5次以上。（必须要优化了）


<img src="http://img.blog.csdn.net/20151103211615749" width = "100" height = "150" alt="图片名称" align=center />


- 调试Hierarchy Viewer 颜色说明					
			
		下方三个原点从左到右：测量、布局、画图时间
		红色：该View所用时间超过大部分View很多
		黄色：该View所用时间超过大部分View
		绿色：该View所用时间低于大部分View
![这里写图片描述](http://img.blog.csdn.net/20151016211507297)

- 引起过度绘制的两个主要因素：层级与背景图片
	
		- 层级为透明时（不添加背景），不会引起过度绘制，但会引起测量、布局、画图时间的显著提高。
		- 改变View形状，也算是绘制一层。添加一个椭圆形的黑色背景，算作两层
		- 值得注意的是，背景图片的绘制是及其耗时的

- 一个通常的错误观念就是使用基本的布局结构(例如：LinearLayout、FrameLayout等)能够在大多数情况下产生高效率的布局。

		基本的线性布局会导致过于累赘的层级嵌套结构。使用相对布局优化。
		但并不是所有情况下都应该用相对布局。（相对布局过于复杂，且通读性差）应考虑权衡关系。

##3.优化措施

- 在Theme中给activity增加背景。使用WindowBackground属性。

		背景的绘制是非常耗时的，在Theme中添加背景，不算绘制一层，并且View渲染时间减少很多。
		具体原因详解请参考：

- 减少层级，没必要的背景图

		（如果一个View和它所在的Layout的颜色相同，就不需要给两个都设置背景）

- 避免使布局太深，而应该让布局更浅更深

		（用相对布局替换线性布局）

- 无用的子节点、父节点删除

		没有免费的午餐，性能优化最重要的一点便是：不要做多余的事情。举例：
		1.想要设置控件之间的间距，使用 layout_marginTop 之类的属性，
		而不是填充一个透明的TextView
		2.如果你需要的效果仅是一张图片加一串文字。那么不需要使用两个控件：TextView+ImageView.  
		TextView一个控件足以。

- 对于要被<include>的布局，如果没有背景或Padding，使用 merge 标签作为根布局

- 避免出现多个使用layout-weight属性的的LinearLayout。

		首先我们必须要承认layout-weight的灵活性，但在使用时，请再三考虑是否真的有必要。
		weight将导致大量的系统开销，每个子项目都要测量两次。

- 合并作为根节点的帧布局(Framelayout)  
		
		你需要知道的一个知识点：Activity或Fragment的默认根布局是FrameLayout。
		如果一个帧布局时布局文件中的根节点，而且它没有背景图片或者padding等。
		更有效的方式是使用<merge />标签替换该< Framelayout />标签 。

- 使用组合控件

		首先说明的是，组合控件并不会减少过度绘制，也不会减少View的绘制时间。
		但它会让你的布局文件看起来非常的清晰。
		并且对于一些条状的控件。类似与下图这样的控件。
		当你需要给这样的控件添加点击事件时，你可能需要给一个layout,两个TextView都添加。
		使用组合控件包装你的view，既符合封装的特性，又可以减少代码量

**重要的东西放到最后说：**

- **重绘控件，提前绘制控件背景与形状，使得View在放到界面上之前就已经画好。极为有效的避免过度绘制。**

		说实话，直接使用原生控件很难避免过度绘制：
		一个Button，继承与TextView，所以直接就已经被绘制了两次（TextView一次，加Button背景第二次）
		那么这是你需要终极绝招：View提前绘制。
		不过需要你抉择的是，提前绘制是一项复杂的工作，所以在复杂布局中使用OK，过于简单的布局就没有必要了。
		楼主正在努力将提前绘制控件类库化，但目前较为遗憾的是，很难抽取提前绘制控件的相同点。
		不同需求画法是不一样的。
		




##4.	案例说明 – 登录界面

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了简化解说，我们使用登录界面作为案例。但对于 控件的提前绘制来说，在登录界面投入和产出并不等比，控件提前绘制，你应该关注复杂的界面，尤其是这复杂的界面上还有动画效果。


开启过度绘制检测后 ,上边是我们未经优化的界面。下边是QQ空间的登录界面。

<img src="http://img.blog.csdn.net/20151103211615749" width = "200" height = "350" alt="图片名称"  />

<img src="http://img.blog.csdn.net/20151016211551825" width = "200" height = "350" alt="图片名称"  />


惊讶吗？没关系，我们也可以做到。

之后再看我们的UI层级图：


![这里写图片描述](http://img.blog.csdn.net/20151016211744054)

下面我们一步步分析优化。


###优化1：

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**优化布局结构，解决过深的UI层级图，与无用的子节点**

- 	过深的LinearLayout嵌套LinearLayout 嵌套LinearLayout 。并且通读性较差

![这里写图片描述](http://img.blog.csdn.net/20151016211802245)

**优化方案1：**采用相对布局，使得布局变浅变宽。最大可以保证只有两层。


**优化方案2：**观察可以得知，布局整体大方向为垂直线性，采用组合控件（带小图标的输入框作为一个整体控件）加垂直线性布局。


- 	无用的子节点。

![这里写图片描述](http://img.blog.csdn.net/20151016211827892)

一个布局里只放了一个Button按钮，次布局为无用的子节点，去掉。


###优化2
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**子view过久的测量时间。**

![这里写图片描述](http://img.blog.csdn.net/20151016211844469)

查看其代码：

	<com.envision.mobile.ui.widget.EditView    
        android:id="@+id/login_name"    
        android:layout_width="0dp"    
        android:layout_height="wrap_content"   
        android:layout_gravity="center_vertical"    
        android:layout_marginRight="8dip"   
        android:layout_weight="1"    
        android:background="@null"       
        android:hint="@string/hint_login_name"    
        android:imeOptions="actionNext"    
        android:paddingBottom="8dip"   
        android:singleLine="true"   
        android:textColor="@color/white"   
        android:textColorHint="@color/white_trans_88"    
        android:textCursorDrawable="@null" />   

**android:layout_weight="1"  属性导致过久的测量时间。 **   


###优化3

在Theme中给activity添加背景。减少一层绘制

		<style name="AppTheme" parent="AppBaseTheme">
		    <item name="android:windowBackground">
			@drawable/bg_homepage
			</item>
		</style>


###优化4：

**重绘控件，提前为控件绘制背景或形状，在控件放到布局上时，就已经被绘制好。**

这里代码量较大，我们放到另一篇文章中讲：
http://blog.csdn.net/u010255127/article/details/49702663

##最终优化效果	
没有红色，最高是绿色
（替换了一些控件和实现方式，不对本文所讲述的内容有影响，我们只看过度绘制检测）


![这里写图片描述](http://img.blog.csdn.net/20151103211244952)



##测试数据

【时间计算】 单位 /ms （测算时间受手机性能影响，数据较为不稳定，需多次测量）
原始状况：

	onCreate 到 onStart ： 381  370 
	onCreate 到 onResume ：383  373
	onStart 到 onResume ： 2     3


在theme加背景

	onCreate 到 onStart ： 274  295
	onCreate 到 onResume ：277  298
	onStart 到 onResume ：  3    3



仅修改布局层次(去掉两个不必要透明布局)

	onCreate 到 onStart ： 272   276
	onCreate 到 onResume ：274  279
	onStart 到 onResume ：  2     3



替换自定义 提前绘制控件

	onCreate 到 onStart ：  289  299  294
	onCreate 到 onResume ：  292 302  296
	onStart 到 onResume ：   3    3    2


结论：我们可以很清晰看到当我们把背景设置到Theme中时，View绘制时间减少的非常明显。
去掉不必要的布局和View，虽然效果甚微，但也减少了。

最后，虽然提前绘制控件并没有起到减少绘制时间的作用，甚至还稍加了一点时间（内存中绘图）。但减少了过度绘制，对界面运行的流畅度起到的作用非常大的。
另外，我们绘制逻辑还有很大优化的空间，这个任重而道远……


最后提一小点，WebView里面的Html页面，过度绘制是检测不到的，也就是说，如果你的Html页面里叠加了100层，那过度绘制检测看起来也是一层。

我所怀疑的是，这一层究竟是表象，只是避开了工具的检测。还是真的可以起到性能优化作用。对WebView研究甚少，还请大神们研究一下。

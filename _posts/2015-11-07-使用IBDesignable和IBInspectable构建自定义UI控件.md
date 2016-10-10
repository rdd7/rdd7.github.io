---
layout: post
title: 使用IBDesignable和IBInspectable构建自定义UI控件
date: 2015-11-07 20:12:09.000000000 +08:00
---

## 怎么突然弄起这个了
之前习惯了代码+frame布局，突然觉得是时候征服一下`StoryBoard`、`Xib`和自动布局了。偶然搜索到CocoaChina上一篇去年的文章（[如何在iOS 8中使用Swift和Xcode 6制作精美的UI组件](http://www.cocoachina.com/industry/20140619/8883.html)），居然可以用`IBDesignable`和`IBInspectable`构建直接在`StoryBoard`和`Xib`上显示和编辑的控件！（原谅我见识浅薄）顿时眼前一亮，于是自己也跟着教程学了一些基础。

## What's this
**IBDesignable**：使用`IBDesignable`去标志一个自定义视图的类（包括 extension 和 category ），IB 将会渲染这个自定义视图在界面中；当你改变这个类的代码时， IB 也会 rebuild 并重新渲染这个自定义视图。

**IBInspectable**：使用`IBInspectable`去标志一个变量，将会使得这个变量在改变时 IB 快速的渲染你的自定义视图，从而达到视图在 IB 界面实时修改的效果。`IBInspectable`不仅可以标志在类里声明的变量，还包括 extension 和 category 里的变量。被标志的变量将会在 IB 的 attribute inspector 面板中出现，和普通的 View 的属性展现形式相同。

## How to play

### 准备
以 iOS 为例，新建一个普通的 iOS 工程，为了尽可能方便，这里新建了一个 Single View Application 。然后在工程中新建一个 Cocoa Touch Class ，这里我新建了一个`MyView`的类，继承自`UILabel`，当然继承其他 View 都是行的，只要是`UIView`或`NSView`的子类就行。工程结构如下图所示：

![工程结构](http://blogsource.oss-cn-hangzhou.aliyuncs.com/IBDesugbable/%E5%B7%A5%E7%A8%8B%E7%9B%AE%E5%BD%95%E7%BB%93%E6%9E%841.png)

### 让IB支持自定义视图类
编辑`MyView.swift`，将里面预写的代码都删除，并在`MyView`类声明前加`@IBDesignable`，如图所示：

	import UIKit
	
	@IBDesignable class MyView: UILabel {
	
	}

这样，这个类就已经可以在IB中实时渲染啦。当然现在去还什么都做不了，还需要`IBInspectable`给我们提供在 IB 上可以修改的属性接口。

### 初始化
首先我们写两个初始化接口

	required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
        text = "init from coder"
        
    }
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        text = "init from frame"
    }

然后转到`StoryBoard`，在 Object Library 中拖出一个 View到 Controller上，选中这个 View 后在右侧的 Identity Inspector 中将其 Class 选择为我们自己的自定义视图类，这里选择为`MyView`，很快在 IB 的界面上我们就看到了自己的自定义视图已经出现了

![](http://blogsource.oss-cn-hangzhou.aliyuncs.com/IBDesugbable/MyView1.png)

从界面上还可以看出，这里是调用了`init(frame)`来初始化的，可能马上有人就反应过来了，从`StoryBoard`实例一个 View ,怎么不是用`init(coder)`呢？其实用模拟器或真机一跑就知道，在运行时调用的的确是`init(coder)`

![](http://blogsource.oss-cn-hangzhou.aliyuncs.com/IBDesugbable/screen.jpg)

所以这个可能出现暗坑的地方自然要多留意一下，对于一些初始化界面的代码，建议统一写到一个私有函数内，然后在上述两个初始化方法中调用，可以免去不少重复代码。这里我写了一个`setup()`函数，将`textAlignment`设为`.Center`,在上述两个初始化方法最后都加上对这个函数的调用，这样比较美观~（-3-)

### 使用IBInspectable

最常用且最能在这里立竿见影的属性，可能真要属`layer`的`cornerRadius`、`borderColor`和`borderWidth`了。

以配置`cornerRadius`为例，我们提供一个直接在 View 层就能访问的`cornerRadius`变量，并通过他的更改影响到`layer`的`cornerRadius`。这一切其实都十分简单，和声明普通的变量一样声明完`cornerRadius`，在他的最前面加上`@IBInspectable`就可以让这个属性在IB的 attribute inspector 界面粗线了！但是显然这样更改这个值还并没有卵用，所以我们要在这个变量的didSet内改变`layer`的`cornerRadius`，**并且让layer的masksToBounds为true**。如此一来，这个`cornerRadius`就可以更改视图的圆角，且在 IB 中能够实时显示效果。`borderColor`和`borderWidth`实现过程相同，具体代码如下：

	@IBInspectable var cornerRadius: CGFloat = 0 {
        didSet{
            layer.cornerRadius = cornerRadius
            layer.masksToBounds = true
            
        }
    }
    
    @IBInspectable var borderColor: UIColor = UIColor.clearColor() {
        didSet{
            layer.borderColor = borderColor.CGColor
        }
    }
    @IBInspectable var borderWidth: CGFloat = 0 {
        didSet{
            layer.borderWidth = borderWidth
        }
    }
    
    
现在去`StoryBoard`里选中之前的自定义视图，在右侧的 Utilities 面板中选中 attribute inspector 面板，会神奇的发现刚才定义的三个属性都已经显示在面板上，修改值会在 IB 界面上动态改变自定义视图

![](http://blogsource.oss-cn-hangzhou.aliyuncs.com/IBDesugbable/CustomViewChange.png)

当时的我很好奇`didSet`既然可以，那`willSet`是否也可以咧，于是我把`borderWidth`的实现代码改成

	@IBInspectable var borderWidth: CGFloat = 0 {
        willSet{
            layer.borderWidth = newValue
        }
    }

发现依然有效，由此可知无论是`willSet`还是`didSet`在 IB 中编辑都是可用的

此外，`IBInspectable`并不是支持所有的数据类型,目前已知可显示在 attribute inspector 面板的数据类型包括`UIImage`,`UIColor`,`Bool`,数字(如`Float`,`Int`,`CGFloat`，不包括`NSnumber`,也不包括枚举类型)，`CGPoint`、`CGSize`、`CGRect`，可能还有遗漏，望指正。

### layoutSubviews()
没错，这个家伙也能在IB编辑视图时被调用,最简单地证明方法时你在这个函数内设置视图的背景为某个颜色，比如红色，然后回去 IB 一看，真变红了。。

这个有什么用呢？当然是为了在设计时，如果自定义视图大小位置改变，内部能快速的做出正确的反应显示在界面上。

举个栗子，在自定义视图中新增一个`ImageView`,并声明一个`IBInspectable`变量`image`，当`image`被设置时`ImageView`的图片同时被设置。在`setup()`函数中初始化`ImageView`并将其加入到自定义视图中,最后在`layoutSubview`中设置其`center`，让其位置一直在 View 的中心点向上**120px**，具体代码如下：

	lazy var imageView = UIImageView()
	@IBInspectable var image: UIImage? {
       	didSet{
           	imageView.image = image
       	}
    }
    private func setup() {
        textAlignment = .Center
        imageView.bounds = CGRectMake(0, 0, 60, 60)
        addSubview(imageView)
    }
    
    override func layoutSubviews() {
        super.layoutSubviews()
        imageView.center = CGPointMake(bounds.midX, bounds.midY - 60)    
    }

回到`StoryBoard`，麻利的设置一张图片，这是无论如何拖动自定义视图，可以看到`ImageView`都能按照我们代码中要求的位置正确摆放。

![](http://blogsource.oss-cn-hangzhou.aliyuncs.com/IBDesugbable/CustomViewImageView.png)

这里也可以看出，对于这个自定义视图内的`ImageView`，是无法在 IB 中直接编辑的。**自定义视图也是IB中最小的编辑单位，如果真要内部的视图同样可在IB中编辑，应该直接在IB中嵌套视图并设置约束。**

### drawRect
(感谢 [@Livinspring](https://disqus.com/by/livinspring/?l=zh) 指正) drawRect方法同样可以在IB中调用并显示，示例代码可以文章最底部在工程文件中找到。


### 继承
和普通的类一样，被标记为`IBDesignable`的类同样可以被继承，且在 IB 上其可编辑的属性依旧可以编辑，子类可以继续新增`IBInspectable`标记的变量在 attribute inspector 面板上。

### 属性更新顺序
如果不知道属性的更新机制与顺序，可能在做依赖其他属性推算某属性的时候出现超出预期的显示结果。在 IB 中，**更改任何一项attribute inspector面板上的属性**，都将会让这个 View **重新将所有的属性初始化并将属性按照属性名的字母升序挨个重新设置一遍值**

修改之前我们使用的代码,新增如下变量和方法：

	@IBInspectable var tmpBool: Bool = true {
        didSet{
            showAttributeAndNumber("tmpBool")
        }
    }
    lazy var count = 0
    
    private func showAttributeAndNumber(funcName: String) {
        text = (text ?? "" ) + String(format:"\n%d  ",count++) + funcName + (tmpBool ? "  true  " : "  false  ")
    }

其余的变量同`tmpBool`一样，在`didSet`内追加`showAttributeAndNumber(funcName)`函数，然后转回`StoryBoard`,看看结果，发现方法已经按照既定要求打印出来,确实是按照属性名的字母顺序升序设置的

![](http://blogsource.oss-cn-hangzhou.aliyuncs.com/IBDesugbable/varibleOrder.png)

更改`tmpBool`的值，发现只有最后一行，即`tmpBool`自己的这一行的值会在打印时改变，其余行的布尔值均为`tmpBool`的默认值。

由于 IB 这个奇怪的设定（可能也想不出有更好的方法了吧。。），有时候确实会导致运行时和编辑时视图的不一致，需要稍微留意一下。

### prepareForInterfaceBuilder() 和 TARGET\_INTERFACE\_BUILDER
`prepareForInterfaceBuilder()`是为了让视图额外准备在 IB 上显示用的。而通过`TARGET_INTERFACE_BUILDER`这个宏定义可以判断当前的编译目标是否为 IB ，从而达到差异化编译的效果。通过两者搭配，可以让`prepareForInterfaceBuilder()`在目标为 IB 时才编译，示例代码如下：

	#if TARGET_INTERFACE_BUILDER
    override func prepareForInterfaceBuilder() {
        //这里写额外为IB准备的视图样式
    }
    #endif
    
特别注意的是，`prepareForInterfaceBuilder()`执行在初始化和各属性被赋值之后，这意味着你在代码里对界面的任何修改都能如实反映在界面上，且你能得到所有属性的最新值（但仍不建议你修改）。

### 一个坑

**不要在构造函数中修改attribute inspector面板中的属性**：在构造函数中如果对 attribute inspector 面板中出现的属性进行了初始化的话，会导致**这些属性在运行时是按照你构造函数中的初始化的值来显示，而在IB编辑界面中却按照 attribute inspector 面板的值来显示**，导致编辑时与运行时界面的不一致。

例如在初始化方法中设置`backgroundColor = UIColor.redColor()`,而在 attribute inspector 面板中`background`的颜色为`whiteColor`，那将会导致在IB中你看到的这个视图背景为白色，而运行时视图背景却为红色。

究其原因也十分简单，在运行`StoryBoard`会通过`init(coder)`构造链创建我们的自定义视图，视图先是设定成了`StoryBoard`上的样式，随后又由于我们追加的初始化代码使得自定义视图又变为我们自己代码定义的样子；而在编辑时，`StoryBoard`先通过`init(frame)`构造链创建自定义视图，生成我们自己代码所定义的样式，而后`StoryBoard`根据 attribute inspector 再逐一给属性赋予新值，于是界面成了 attribute inspector 上所定义的属性的样子。

### 很潦草的总结
+使用`IBDesignable`标记一个类(包括extension 和 category )使其被 IB 认为是可以在 IB 上编辑的

+使用`IBInspectable`标记一个变量，使其可以被显示在 IB 的 attribute inspector 面板上

+被`IBInspectable`标记的变量其值在 IB 面板中改变时，`willSet`和`didSet`均按照其自然顺序依次调用

+在 IB 的属性窗口已经列举出的属性建议直接在 IB 中修改，不建议在代码初始化中再修改，因为这样会使得在编辑时与运行时的界面产生不统一，这种问题真要日后找起来，那简直是。。。。

+如果属性的修改与显示会依赖其他属性，需要留意IB对属性的更新顺序：**更改任何一项attribute inspector面板上的属性，都将会让这个View重新将所有的属性初始化并将属性按照属性名的字母升序挨个重新设置一遍值**


### 这个工程
文中使用的工程源码已上传Github，[点此下载](https://github.com/rdd7/IBComponent)

（本文由[Rdd7](http://rdd7.com)原创，转载请指明[出处](http://rdd7.com/2015/11/使用IBDesignable和IBInspectable构建自定义UI控件/)）

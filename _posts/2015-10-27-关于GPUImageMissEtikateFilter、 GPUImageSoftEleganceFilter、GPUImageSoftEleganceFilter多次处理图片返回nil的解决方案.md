---
layout: post
title: 关于GPUImageMissEtikateFilter、 GPUImageSoftEleganceFilter、GPUImageSoftEleganceFilter多次处理图片返回nil的解决方案
date: 2015-10-27 15:32:24.000000000 +08:00
---

最近工程因为涉及到滤镜的应用而用到了**GPUImage**库，其自带的一些滤镜使用起来可谓十分方便，但是在使用**GPUImageAmatorkaFilter**、**GPUImageMissEtikateFilter**和**GPUImageSoftEleganceFilter**时却遇到了奇怪的问题，这三个滤镜可以正确的在图片预览栏处理图片，却不能在图片的主要视图处理，且生成的图片全为**nil**。

对于这个问题原因的产生，必须先说说自己项目中滤镜的组织结构和使用方式，大致结构如下所示：


![image](http://blogsource.oss-cn-hangzhou.aliyuncs.com/GPUimage/%E5%A4%84%E7%90%86%E5%9B%BE.jpg)

如上图所示，滤镜统一放在一个滤镜数组中，无论是生成主视图图片还是生成工具栏预览图片，所用的滤镜均来自同一个滤镜数组。于是这里就产生了一个问题：在工具栏的图片都能被各个滤镜渲染成功，而到了主视图时，**GPUImageAmatorkaFilter**、**GPUImageMissEtikateFilter**和**GPUImageSoftEleganceFilter**将不能正确渲染，且生成空文件。

对于这个坑查了诸多资料，终于在Github上找到了原因：[https://github.com/BradLarson/GPUImage/issues/1522](https://github.com/BradLarson/GPUImage/issues/1522)，其中[se0lus](https://github.com/se0lus)指出了这个问题的原因，大致意思是由于上述三个滤镜均是使用了**GPUImageLookupFilter**，而**GPUImageLookupFilter**是**GPUImageTwoInputFilter**的子类，**GPUImageTwoInputFilter**需要两张图片才能工作，一般一张为需要渲染的照片，另一张可以认为是用于渲染目标图片的图片（**LookupImage**），当两张图片都到齐后进行处理并输出图片。**GPUImageAmatorkaFilter**等滤镜在初始化时会将**LookupImage**作为第一个输入源，所以在第一次接收到一张图片时滤镜能正常工作，之后当第二次调用时，由于只有一个输入源，所以滤镜工作不正常。

回顾这个项目，一切就都说得通了。由于所有的滤镜都是从一个滤镜数组中取出的，在底部预览工具栏上每个滤镜均被调用了一次并输出了对应渲染后的图片，此时滤镜数组中的**GPUImageAmatorkaFilter**、**GPUImageMissEtikateFilter**和**GPUImageSoftEleganceFilter**已经被调用过一次，随后点击这几个滤镜对应的图片打算渲染主视图的图片时就出错了，因为这个滤镜当前只有一个输入源了，**LookupImage**已经不在另一个输入源，于是出现了生成图片失败。

对于这个问题，[se0lus](https://github.com/se0lus) 在解答中给出了解决方案，但是涉及到修改GPUImage库的源文件，对于使用cocoaPods进行管理而言实在太心累，既然只要一个全新的滤镜就能解决问题，在条件允许的情况下完全可以再创建一个滤镜，在我这个项目中由于传递的都是滤镜的实例，所以能够很轻易的创建出对应的新的实例，并将当前新的滤镜实例插入到对应的GPUImageFilterPipeline中最后渲染即可轻松解决：


	- (void)setFilter:(GPUImageFilter *)filter{
	
    	id baseNewFilter = [[[filter class] alloc]init];
    
    	[self.usedFilters replaceFilterAtIndex:baseFilterIndex withFilter:baseNewFilter];
    
    	[self.picture processImage];
    
	}
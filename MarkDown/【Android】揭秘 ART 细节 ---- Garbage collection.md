# 【Android】揭秘 ART 细节 ---- Garbage collection
##背景
　　Dalvik ：http://zh.wikipedia.org/wiki/Dalvik%E8%99%9A%E6%8B%9F%E6%9C%BA
　　ART ：http://source.android.com/devices/tech/dalvik/art.html
　　
---
Ian Rogers  在Google IO 2014上讲述了 The ART runtime 的Garbage Collection部分，通过他的讲述，我们可以了解到ART在垃圾回收方面有哪些改进的地方。开门见山，下面我们就来了解一下具体的细节：

首先来看一下GC在Dalvik里是如何工作的：
![此处输入图片的描述][1]


                           图1 :  GC的过程
![此处输入图片的描述][2]                           

                           图2 : 挂起所有线程进行标记，垃圾回收以释放空间    
                           
从图1可以看到当Dalvik开始垃圾回收时，GC会去查找所有活动的对象，这个时候整个程序的线程就会挂起，并且虚拟机内部的所有线程也会同时挂起(图2) ，这样目的是在较少的堆栈里找到所引用的对象。需要注意的是这个回收动作是和应用程序同时执行。这里之所以要挂起所有线程是确保所有程序没有进行任何变更，与此同时GC会隐藏所有处理过的对象，最终，确保标记了所有需要回收的对象后，GC才会恢复所有线程，并释放空间。因此在Dalvik里，挂起所有线程这个动作的优先级非常高，在内存紧张的时候就会频繁执行这个动作，这样就会造成丢帧，界面卡顿的现象。  
![此处输入图片的描述][3]

                        图3 ： 为什么Dalvik 里的GC这么挫？
从图3可以看到，当发现需要给一个较大的对象(蓝色方块)分配空间时，发现可用空间还是够的，但没有这么大的连续空间供新对象使用，这个时候就不得不进行一次GC回收（红色方块）（图3），为大对象腾出较大并且连续的空间。这就是我们在分配一个较大对象的时候非常容易引起丢帧和卡顿的原因之一。解决方案可以是:把较大对象分解成几个较小的对象再进行初始化，但这解决不了根本问题。

　　上图我们还可以用一个现实中比较形象的小区停车现象来阐释:一辆较长的汽车A（蓝色方块）来找车位，发现空的位置很多，但车与车之间的间距较大，没有适合A汽车停的位置，这个时候就不得不让车位管理员M（GC）去查找确认是否有非本小区的车辆（红色方块）并回收车位，供A汽车使用。（这里我们可以发现，如果车停得够紧凑，就无需麻烦车位管理员）


　　通过上面3张图我们可以看到Dalvik中GC的问题如下:

　　1. GC时挂起所有线程

　　2. 大而连续的空间紧张

　　3. 内存碎片化严重

　　下面我们来了解一下ART是如何解决这些问题的：  
　　![此处输入图片的描述][4]
　　
                        图4 在ART中不需要挂起所有程序的线程
                        
这里可以对比着图1一起看，在ART中GC会要求程序在分配空间的时候标记自身的堆栈，这个过程非常短，不需要挂起所有程序的线程.这样就节约了很大一部分时间去查找活动对象。（解决问题1）
![此处输入图片的描述][5]

                        图5  提供 LOS ：large object space 专供Bitmap使用
                        
从图5可以看到，ART里会有一个独立的LOS供Bitmap使用，从而提高了GC的管理效率和整体性能。

　　同样我们从小区停车现象理解:小区里划出了一块大车专用的区域，使得大车省去了找车位的时间，也减少了通知管理员M（GC）的次数。（解决问题2）
　　![此处输入图片的描述][6]
　　
　　                    图6 ART中的 moving collector
　　                    
在ART里还会有一个moving collector来压缩活动对象(绿色方块)，使得内存空间更加紧凑。

　　从小区停车现象理解：车位管理员M会定期移动停得不规范的车，使得停车空间更加紧凑，最大化利用有效空间。（解决问题3）

　　在解决了以上三个问题之后，ART就具备了以下优点:

　　1.更少的内存碎片

　　2.更短更少的中断和阻塞

　　3.更低的内存使用率

 

　　总结 ：Google在ART里对GC做了非常大的优化，从演示的数据里看，内存分配的效率提高了10倍，GC的效率提高了2-3倍。主要是通过标记时机的变更使中断和阻塞的时间更短；通过LOS解决大对象的内存分配和存储问题；通过moving collector来压缩内存，使内存空间更加紧凑，从而达到GC整体性能的巨大提升。　　                    

 
  


  [1]: https://raw.githubusercontent.com/Duncan-Liang/Android-Runtime/master/MarkDown/pic/%E5%9B%BE1_GC%E7%9A%84%E8%BF%87%E7%A8%8B.png
  [2]: https://raw.githubusercontent.com/Duncan-Liang/Android-Runtime/master/MarkDown/pic/%E5%9B%BE2_%E6%8C%82%E8%B5%B7%E6%89%80%E6%9C%89%E7%BA%BF%E7%A8%8B%E8%BF%9B%E8%A1%8C%E6%A0%87%E8%AE%B0%EF%BC%8C%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E4%BB%A5%E9%87%8A%E6%94%BE%E7%A9%BA%E9%97%B4.png
  [3]: https://raw.githubusercontent.com/Duncan-Liang/Android-Runtime/master/MarkDown/pic/%E5%9B%BE3_%E4%B8%BA%E4%BB%80%E4%B9%88Dalvik%20%E9%87%8C%E7%9A%84GC%E8%BF%99%E4%B9%88%E6%8C%AB%EF%BC%9F.png
  [4]: https://raw.githubusercontent.com/Duncan-Liang/Android-Runtime/master/MarkDown/pic/%E5%9B%BE4_%E5%9C%A8ART%E4%B8%AD%E4%B8%8D%E9%9C%80%E8%A6%81%E6%8C%82%E8%B5%B7%E6%89%80%E6%9C%89%E7%A8%8B%E5%BA%8F%E7%9A%84%E7%BA%BF%E7%A8%8B.png
  [5]: https://raw.githubusercontent.com/Duncan-Liang/Android-Runtime/master/MarkDown/pic/%E5%9B%BE5_%E6%8F%90%E4%BE%9BLOS_large%20object%20space%20%E4%B8%93%E4%BE%9BBitmap%E4%BD%BF%E7%94%A8.png
  [6]: https://raw.githubusercontent.com/Duncan-Liang/Android-Runtime/master/MarkDown/pic/%E5%9B%BE6%20ART%E4%B8%AD%E7%9A%84%20moving%20collector.png
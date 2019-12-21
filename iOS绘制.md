## iOS图形绘制整理

常规iOS App开发一般只需要用好基于OpenGL的Core Graphics、Core Animation、Core Image即可，一般只有游戏、相机、视频类应用会需要跟OpenGL或者Metal直接打交道的需求。

UI优化的核心就是平衡CPU和GPU的负载。虽然texture composition这种工作在GPU上完成的速度要远高于在CPU上的速度，但CPU还可以完成坐标大小计算之类的工作。当GPU负载低时，可以将更多绘制工作交给GPU，当GPU负载高时，可以将一部分绘制工作分摊给CPU来完成。

#### 绘制过程

60fps的屏幕硬件每1/60秒会发送一个同步信号给GPU，GPU从VRAM（显存）中取出这一帧的texture进行合成（composition）放到屏幕buffer中进行显示。因此这里有两个限制，一是VRAM的容量限制，即只能保存一定数量的texture，二是GPU合成速度限制，16.7ms内能完成的计算是有硬件极限的。

要展示一张PNG或者JPEG图片，首先CPU要进行解压图片文件然后像素化，像素化后的数据经由CPU和GPU之间的总线传输到VRAM中，有大量texture的复杂界面耗时自然也会稍久一些。当屏幕上已有的texture只是发生了位置变化，CPU只需要把新的位置告诉GPU即可，而无需再进行重新像素化、传输。

#### GPU Composition

合成（像素化）的原理，就是把重叠的layer的对应像素颜色根据blend公式计算出最终这个硬件像素该显示的颜色。假设两层layer叠加时，最简单的blend公式就是R = S + D * （1 - Sa），S表示上层layer的RGB值（预乘alpha），D表示下层layer的RGB值（同样预乘了alpha），Sa是上层layer的alpha值，每两层layer混合就有这样一次计算，多层layer混合就要进行多次这样的计算。由此公式可以发现，当最上层layer的Sa = 1时，R = S，底下的layer合成其实就无需计算了，该像素点的最终颜色就是S。但GPU在合成之前没办法知道上层layer是不是每个像素的alpha通道都是1，知道这一点的是我们，programmer。所以这就是`CALayer`会有个`opaque`属性的原因，在可能的情况下使用`opaque = true`的layer，能为GPU节省很多工作。

在上面的像素合成中，我举的例子其实是像素对齐的，也就是说每层layer的像素都刚好落在一个真正的屏幕像素上。而如果某层layer没有像素对齐，比如`origin = {0.25, 0.25}`这种奇葩layer，在合成它所在的每一个屏幕像素时，都要取它的好几个像素来计算该屏幕像素的值，显然这也会增加合成的计算量。Instruments中color misaligned images功能就是用来帮你找出没有像素对齐的layer的，尽量保证像素对齐。

#### Offscreen Rendering

离屏渲染有时候是我们代码里写的，有时候是Core Animation自动触发的。所谓离屏渲染，就是先合成一部分layer到一块buffer（非屏幕显示的buffer）中缓存起来，离屏二字的本质就是使用了非屏幕显示的buffer。当这些layer的内容没有发生变化，自然就不用在每一次同步信号到来时都重新合成，可以直接重用缓存buffer中的内容。`CAlayer`的`shouldRasterize`属性的就是是否开启缓存用的，对于GPU来说，对layer开启缓存意味着要多做一些额外工作，开启是否有性能提升需要你自行判断。Instrument中的Color Offscreen-Rendered Yellow可以帮助你发现哪些layer是离屏渲染的，而选中Color Hits Green and Missed Red选项，你就可以发现哪些layer的离屏渲染buffer被重用了，哪些没有。

通常情况下，我们都想避免离屏渲染，毕竟相比直接渲染到屏幕显示buffer中，开销会更大。但离屏渲染并不总是坏事，只要有的layer内容不变，能够多次重用，也许反而会提升性能。

使用`mask`、`corner radius`、`shadows`都会造成离屏渲染。对于那种矩形图片加圆角`mask`的场景，可以预先把圆角效果加到图片上，然后缓存加了圆角效果后的图片。但如果是加矩形`mask`，可以直接使用`contentsRect`属性，没必要缓存。

#### CPU和GPU的负载均衡

在iOS中Core Graphics的API是基于CPU的，Core Animation则封装了OpenGL的API，同时也会整合Core Graphics的绘制结果。Instrument中的OpenGL ES Driver可以观察GPU的使用率，尽可能多得压榨GPU，因为在渲染的工作上，它的性能更强同时也更省电（这对移动设备非常关键）。只有在GPU使用率较高时并且拖累了fps时，再考虑分摊一些工作给CPU。

#### 像素格式

像素存放的格式有很多种，常见的有ARGB、xRGB、RGBA、RGBx，x表示不存在alpha通道，但因为内存对齐的原因，会放着占位。ARGB又可以分为32bpp&8bpc，64bpp&16bpc，128bpp&32bpc，bpp表示bits-per-pixle，bpc表示bits-per-componet，每个像素使用的bits越多，图片的颜色过渡会更平滑。

在内存中并不一定是一个像素接着一个像素排列的，有时候会把每个像素的R、G、B、A抽取出来分成四大块存放。

当处理视频时，YCbCr是很常见的像素格式，这种格式是近似于人类肉眼看东西的效果，对Y最敏感，对Cb、Cr不那么敏感，因此在压缩时Cb、Cr可以被压缩的更狠。YCbCr和RGB之间通过公式换算。

#### 图片格式

JPEG、PNG等都是图片压缩后的格式。要从磁盘显示一张JPEG图片，需要先加载图片内容到内存，然后CPU对其进行解压像素化，然后才是渲染到屏幕上。如果在`UITableView`中每个`cell`都要加载一张未解压的图片，相信我，你会觉得自己在看幻灯片。

JPEG的压缩是有损的，会摒弃对人眼来说不那么敏感的信息，在压缩照片上JPEG的压缩率会很高；但对于颜色变化迅速的，比如黑底白字的网页，压缩率就很一般，这种图片被JPEG压缩后的信息丢失甚至可能非常明显。PNG压缩是无损的，论压缩照片是不及JPEG的，但对于颜色变化迅速的，PNG则更加适用。而且PNG的解压过程也会比JPEG的解压来的简单一些。

#### 并发绘制

众所周知，UIKit的类只能在主线程使用。但如果在`draw(_ rect:)`中要做大量的自定义绘制工作，就可能会超过16.7ms，造成掉帧。自然而然，我们想到了concurrent。可是在这个函数中获取的`CGContextRef`，我们又必须在主线程中往里画东西，咋办呢？我们可以在其他线程创建自己的bitmap context，因为context是线程独立的。绘制完成后再回到主线程把绘制结果设置为对应的layer的内容，比如UIImageView的UIImage。

要注意的坑是，有时候在子线程完成绘制后，界面上可能已经不需要绘制结果了。比如用户马上就退出了页面，或者`UITableViewCell`已经被重用了。

还有些时候主线程并没有等着接收绘制内容的控件，可以考虑打开`CALayer`的`drawsAsynchronously`，它会把`draw(_ rect:)`中对Core Graphics的调用记录下来，然后在子线程中调用。是否对fps有提升则需要测试。

#### CALayer相关的tricks

如果直接把`CALayer`的`contents`设置为图片，那么Core Animation会直接把图片的像素数据当作纹理上传给GPU的VRAM。如果使用纯色layer，不要设置`contents`，直接修改`backgroundColor`，CPU就不需要上传那么多像素数据，而只需要把设置的颜色传给GPU，剩下的工作GPU会来完成。`CAGradientLayer`也是类似，只会把渐变的位置和颜色等信息传给GPU即可。
灰度图的几种算法资料

只看有哪些种算法的话, 看大字和图就行了. 

第一篇原文在此, 没有完全翻译, 有些作者的吐槽什么的就略过了.
http://www.tannerhelland.com/3643/

第二篇只介绍了三种GIMP用的, 作为对第一篇的补充插进去了. 
http://www.johndcook.com/blog/2009/08/24/algorithms-convert-color-grayscale/

# Seven grayscale conversion algorithms (with pseudocode and VB6 source code)

## 灰度图 - 介绍

黑白照片(http://en.wikipedia.org/wiki/Monochrome_photography)可以追溯到19世纪中期以前. 即使有了彩色照片黑白照片也还是很流行. 数字革命使黑白照片更流行了因为任何一台数码相机都可以拍摄黑白照片(模拟相机需要特殊胶片). 黑白摄影有时被认为是摄影艺术中的"雕塑". 它抽象主题, 使得拍摄者专注于形式和解释(form and interpretation, 这是艺术术语吗? )而不是简单的复制真实场景. 

因为"黑白"这样的术语并不精准 - 黑白照片其实是有很多不同的灰色构成的 - 本文将这样的图片称为"灰度图"(大家都是这么叫的好么). 

我还要用到几个术语. 一个是"color space". color space是可视化所有颜色的方法. 不一样的颜色表达有不一样的color space. RGB color space(http://en.wikipedia.org/wiki/RGB_color_space)是一个立方体, HSL (http://en.wikipedia.org/wiki/HSL_color_space)是锥形, 或者两个锥形, 或者圆柱形, YIQ和YPbPr有更多的抽象形状. 本文讨论前两个. 

还会用到颜色通道(color chanel, http://en.wikipedia.org/wiki/Color_channel). 多数数字图像使用3个颜色通道: 红绿蓝. 把三个颜色通道的图像叠加在一起就是原图了. 不同的颜色模型有不同的通道(有时是颜色, 有时是亮度, 饱和度之类的), 本文着重于RGB通道. 

## 灰度图算法基本上都是这么算的

所有的灰度图算法都有下面三步: 

1. 取得像素RGB分量
2. 使用奇妙的数学公式把三个颜色分量编程一个灰度值
3. 把RGB三个颜色分量都换成这个灰度值

描述灰度图算法时都是在讲第二步 - 都是数学. 所以你看到这样的公式时:

<pre><code>
Gray = (Red + Green + Blue) / 3
</code></pre>

要知道这个算法实现起来是这样的: 

<pre><code>
For Each Pixel in Image {

   Red = Pixel.Red
   Green = Pixel.Green
   Blue = Pixel.Blue

   Gray = (Red + Green + Blue) / 3

   Pixel.Red = Gray
   Pixel.Green = Gray
   Pixel.Blue = Gray

}
</code></pre>

## 原图

![alt text](http://www.tannerhelland.com/wp-content/uploads/Secret_of_Monkey_Island-600x375.jpg "This bright, colorful promo art for The Secret of Monkey Island: Special Edition will be used to demonstrate each of our seven unique grayscale algorithms.")
This bright, colorful promo art for The Secret of Monkey Island: Special Edition will be used to demonstrate each of our seven unique grayscale algorithms.

## 方法一 - Averaging (平均算法, 又快又差)

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_average-600x375.png "Grayscale image generated from the formula: Average(Red, Green, Blue).")
Grayscale image generated from the formula: Average(Red, Green, Blue)

"Averaging"是用的最多的灰度图算法, 算法这样: 

<pre><code>
Gray = (Red + Green + Blue) / 3
</code></pre>

简单, 快 - 新手的不二之选. 这个公式的灰度图也挺好看的, 实现简单易于优化(快查表效果不错). 缺点呢 - 虽然又快又容易, 它在用灰度对阴影的表现和人眼感知明度(luminosity, brightness)的方式上做的不好. 所以我们需要其他的方法. 

## 方法二 - Correcting for the human eye人眼矫正

(也叫"luma"或者"luminance", 但是这个术语并不精准<http://poynton.com/PDFs/YUV_and_luminance_harmful.pdf>)

![alt text](http://www.tannerhelland.com/3643/grayscale-image-algorithm-vb6/grayscale_humaneye/ "Grayscale generated using a formula similar to (Red * 0.3 + Green * 0.59 + Blue * 0.11)")
Grayscale generated using a formula similar to (Red * 0.3 + Green * 0.59 + Blue * 0.11)

肉眼不太看得出两张图的区别. 看下下面的图, 方法一"Average method"用在上边, 方法二用在下边: 

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_average_and_humaneye-600x375.png "If you look closely, you can see a horizontal line running across the center of the image. The top half (the average method) is more washed-out than the bottom half. This is especially visible in the middle-left segment of the image, beneath the cheekbone of the background skull.")
If you look closely, you can see a horizontal line running across the center of the image. The top half (the average method) is more washed-out than the bottom half. This is especially visible in the middle-left segment of the image, beneath the cheekbone of the background skull.

用文末附带的源码可以看得更清楚. 

方法二利用了人眼锥形体密度对不同颜色不一样的原理. 人眼对绿色的感知强过红色, 红色强过蓝色. 这一点是真的 - 解释了一通. 

因为人眼对颜色的感知不平均, "平均法"就不准确了. 好的灰度算法会根据人眼对每个颜色的感知来权衡颜色, 而不是平均分配. 下面是个常用的公式(Photoshop, GIMP): 

<pre><code>
Gray = (Red * 0.3 + Green * 0.59 + Blue * 0.11)
</code></pre>

(注: 参考二给的公式有争议)
(再注: http://blog.csdn.net/zyl910/article/details/749752 这篇文章给了很好的优化算法)

很惊讶三种颜色权重差距如此之大吧? 这个公式计算量大一点, 但是生成的灰度图更有动感. 看源代码. 

值得一提的是, 哪个灰度图公式是最好的并无定论. 在源码中我使用原始的ITU-R的推荐(BT.709), 历史的先例. 我用的公式这样子, 叫做"Luma": 

<pre><code>
Gray = (Red * 0.2126 + Green * 0.7152 + Blue * 0.0722)
</code></pre>

有些现代数字图像和视频格式使用BT.601, 在系数上有一点不同:

<pre><code>
Gray = (Red * 0.299 + Green * 0.587 + Blue * 0.114)
</code></pre>

本文不讨论哪种更好. 拓展阅读推荐The work of Charles Poynton (http://poynton.com/). 对99%的程序员来说二者的区别无关紧要. 哪个都比"平均法"强. 

## 方法三 - Desaturation (冲淡颜色, 我更愿意叫成去饱和度)

(参考二管同一个公式叫lightness method)

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_desaturate-600x375.png "A desaturated image. Desaturating an image takes advantage of the ability to treat the (R, G, B) colorspace as a 3-dimensional cube. Desaturation approximates a luminance value for each pixel by choosing a corresponding point on the neutral axis of the cube.")
A desaturated image. Desaturating an image takes advantage of the ability to treat the (R, G, B) colorspace as a 3-dimensional cube. Desaturation approximates a luminance value for each pixel by choosing a corresponding point on the neutral axis of the cube.

描述一个像素颜色的方式有多种. 多数程序员使用RGB颜色模型. 对机器来说这种描述方法很好但是对人类来说很难具象. 一些话引入HSL模型. HSL表示hue(颜色), saturation(饱和度)和lightness(亮度). 见维基百科. 

对图片Desaturation要先把RGB颜色转换成HSL, 然后把饱和度置为0. 简单来讲就是把一个颜色的饱和度减到最小. 推导很复杂, 结论很简单. 要把一个像素去饱和度只需要取Max(R,G,B)和Min(R,G,B)的平均值就好了. 

<pre><code>
Gray = ( Max(Red, Green, Blue) + Min(Red, Green, Blue) ) / 2
</code></pre>

维基百科会告诉你为什么. 

Desaturation会生成一个单调柔和的灰度图. 和方法二的人眼矫正图对比, 你会注意到二者的对比图不同. 方法二更像Ansel Adams的照片<http://www.anseladams.com/category_s/71.htm>, 去饱和度更像便宜的傻瓜相机拍出来的. 目前三种方法Desaturation的图像最单调(低对比度)也最黑. 

## 方法四 - Decomposition (解构)

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_decompose_max-600x375.png "Decomposition using maximum values")
Decomposition using maximum values

![alt text](http://www.tannerhelland.com/3643/grayscale-image-algorithm-vb6/grayscale_decompose_min/ "Decomposition using minimum values")
Decomposition using minimum values

解构图像可以看做去饱和度的简单版. 解构图像, 我们取每个像素的灰度值位三个颜色通道的最大或最小值. 注意是每个像素, 我不知道这里作者有什么好解释的说了好多. 

最大值解构: 

<pre><code>
Gray = Max(Red, Green, Blue)
</code></pre>

最小值解构: 

<pre><code>
Gray = Min(Red, Green, Blue)
</code></pre>

可以想见一个很黑一个很亮. 这个方法对我要做的事情没有用处. 

## 方法五 - 单色通道

顾名思义就是取一个颜色通道的值作为灰度值. 作者讲的太罗嗦. 

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_rechannel_red-600x375.png "Grayscale generated by using only red channel values.")
Grayscale generated by using only red channel values.

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_rechannel_green-600x375.png "Grayscale generated by using only green channel values.")
Grayscale generated by using only green channel values.

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_rechannel_blue-600x375.png "Grayscale generated by using only blue channel values.")
Grayscale generated by using only blue channel values.

算法描述太简单了不翻译了. 

多数数码相机的"灰度图"模式用的都是这个算法<http://en.wikipedia.org/wiki/Charge-coupled_device#Color_cameras>. 用绿的. 这个方法多半也不会采用吧. 

## 方法六 - Custom of gray shades 阴影法?

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_4_shades-600x375.png "Grayscale using only 4 shades - black, dark gray, light gray, and white")
Grayscale using only 4 shades - black, dark gray, light gray, and white

这个方法可能黑白图才会用到. 

这个方法有意思. 我是从方法六开始写这个项目的, 让用户指定有多少阶灰度可以用. 2到256都行. 2就是黑白图, 256就是方法一的结果. 项目采用8位颜色通道, 但16位和24位的一样工作. 

算法选择X#个灰度值, 在0, 黑色~255白色之间等分. 上图用了四个灰度. 下图用了16个: 

![alt text](http://www.tannerhelland.com/3643/grayscale-image-algorithm-vb6/grayscale_16_shades/ "In this image, we use 16 shades of gray spanning from black to white")
In this image, we use 16 shades of gray spanning from black to white

这个灰度算法稍微复杂点, 长这样: 

<pre><code>
ConversionFactor = 255 / (NumberOfShades - 1)
AverageValue = (Red + Green + Blue) / 3
Gray = Integer((AverageValue / ConversionFactor) + 0.5) * ConversionFactor

Notes:
-NumberOfShades is a value between 2 and 256
-technically, any grayscale algorithm could be used to calculate AverageValue; it simply provides
 an initial gray value estimate
-the "+ 0.5" addition is an optional parameter that imitates rounding the value of an integer
 conversion; YMMV depending on which programming language you use, as some round automatically
</code></pre>

作者觉得这个算法很艺术. 附件源代码实时渲染各种灰度图, 所以可以比较一下. 

## 方法七 - Custom # of gray shades with dithering

散乱抖动吗? 例子里面用水平误差分散抖动error-diffusion dithering

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_4_shades_dithered-600x375.png "This image also uses only four shades of gray (black, dark gray, light gray, white), but it adds full error-diffusion dithering support")
This image also uses only four shades of gray (black, dark gray, light gray, white), but it adds full error-diffusion dithering support

这个算法奇怪. 如同前面的方法, 用户指定2到256的一个值然后算法计算灰度的分布. 但是这个算法加了抖动支持. 

Dithering的解释. 在图像处理中dithering利用光学成像原理让图像看起来更多彩. Dithering算法把颜色用另一种形式分散 - 有序或者任意 - 让人眼以为它接收到了比实际更多的颜色. 不明白的话看看这个: <http://en.wikipedia.org/wiki/Dithering#Algorithms>

抖动算法有很多. 作者提供的是比较简单的误差分散机制: 水平方向一维分散颜色. 

上图其实只有四种颜色 - 黑色, 深灰色, 浅灰色和白色 - 但是因为混在一起, 从远看比方法六中同样只使用四个颜色的图片看起来丰富多了. 下面比较一下: 

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_4_shades_sidebyside-600x375.png "The left side of the image is a 4-shade non-dithered image; the right side is a 4-shade image WITH dithering")
The left side of the image is a 4-shade non-dithered image; the right side is a 4-shade image WITH dithering

可用的颜色不多时, 抖动算法保留了图片更多的细微差别, 代价就是图片看起来脏乱斑驳. 有些抖动算法好一些; 作者用的比较折中. 

最后举个栗子, 下面是16色抖动灰度图, 然后是一个抖动图和不抖动的比较. 

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_16_shades_dithered-600x375.png "Hard to believe only 16 shades of gray are used in this image, isn't it?")
Hard to believe only 16 shades of gray are used in this image, isn't it?

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_16_shades_sidebyside-600x375.png "As the number of shades of gray in an image increases, dithering artifacts become less and less noticeable. Can you tell which side of the image is dithered and which is not?")
As the number of shades of gray in an image increases, dithering artifacts become less and less noticeable. Can you tell which side of the image is dithered and which is not?

这部分的算法比较复杂, 见Grayscale.frm中方法drawGrayscaleCustomShadesDithered 的实现. 有注释. 

## 结论

如果你的网速慢的话可能加载这些图片会遇到一些困难. 

对工程的讲解, 源码, 界面. 很简单的. 

![alt text](http://www.tannerhelland.com/wp-content/uploads/grayscale_source_code_GUI-600x344.jpg
 "GUI of the provided source code. The program also allows you to load your own images.")
GUI of the provided source code. The program also allows you to load your own images.

工程链接和说明. 
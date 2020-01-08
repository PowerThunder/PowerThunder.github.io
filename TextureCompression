# 手游性能优化之深入理解Texture Compression
一、引子

　　手游项目开发日常里，经常有美术同学搞不清Photoshop制图软件与Unity3D游戏引擎之间的图片assets流转逻辑，在工作输出时经常出现如下疑问：

1、要JPG的，还是要PNG的？

2、JPG的要压存为多高质量的？

3、PNG的还要压？引擎不是自动处理的么？

4、为毛非要正方形的？我这个图实在是没法儿做方的怎么弄？

5、图太大，要选哪个压缩方式？有的怎么选了也没效果？有的又压的太糊！

6、这个效果不行，开发没有还原好啊！

　　所以开发或者技美同学要经常解释这些问题，这里面的确有些内容比较难以说明白，一个是厂商比较多流派，另一个是有些知识点（技术历史）需要厘清，本文的作用就在于此，但不会很全面。

　　下文中出现Texture的地方均指代在Unity3D场景下。

 

二、不要混淆JPG/PNG等图片压缩格式与Texture Compression

JPG/PNG是变长编码格式variable bit length，它有个特点：如下图所示，颜色变化少频率低的部分，编码后占的内存字节数就少。
![image](http://gameweb-img.qq.com/gad/20160324/09ddbd4243cf45b7bc73ef4459ce71ae.001.1458789080.jpeg)
　　所以，编码后长度变来变去带来的坏处就是：无法准确的计算出原图一个坐标处的color对应的压缩到了哪里？除非你把整张图片都解压完毕，如下图所示。







　　这一致命弱点直接导致了GPU 里的Texture Sample算法机制不可用，所以Unity3D引擎里也不会直接使用JPG/PNG这种编码格式来打包图片assets资源。

结论：JPG/PNG是用来在游戏制作流程中间传递美术内容的，最终在游戏引擎里需要转变成一种固定码率的、可寻址的流式压缩格式，以方便随机寻址和采样。

 

三、PVRTC/ETC等Texture Compression格式，直接被GPU读取到显存，用时无需解压

　　是的，无需解压！美术同学可能难以理解，JPG压缩了之后不解压你怎么看？对，GPU就是这么屌！下面举个例子：

假设一张4M的RGB三通道JPG图片，经过ETC1压缩后变为1M，GPU 里的Fragment渲染模块根据当前的Render State决定去加载Texture Buffer里对应的某一块字节数据，然后经过实时的运算来还原出相应的颜色值，即采样过程，这中间便省去了将整个图片解压还原出来的步骤，因为GPU芯片很擅长这种固定的算法。

好处显而易见：1. 支持随机采样，不用把整张图load进内存； 2. 即使整个load进内存，也不用解压展开成4M；3.如果你嫌1M还是太大，还能在打包游戏时用ZIP再压缩一把；

 

四、PVRTC 2bpp/4bpp, ETC1, ETC2,  DXT, ASTC 这些都是什么鬼？

是的，手机市场就是这么乱。GPU芯片提供商有Imagination, ARM, NVIDIA, QualComm，各有各的芯片系统(SoC) IP，我们搞软件开发的要细究起来恐怕会吐血身亡，毕竟是不同的行业。不管那么多，结论有：

1、Imagination是被Intel和Apple两大头持股的，iOS平台得优选他家的PVRTC压缩格式；

2、ETC1基本上是Android采用的公案，所以选它没错，但仅支持RGB三通道，如果还有A通道，得另行单独压一张图；

3、ETC2虽然升级了，但目前的主流OpenGL ES 2.0规格尚不支持，不要多想了；

 

五、浅析ETC1 --- 一个像素2个bits是怎么做到的？

ETC stands for Ericsson Texture Compression and is an open standard supported in OpenGL and OpenGL ES. The technique allows lossy compression of images at a ratio of 4:1 (depending on input format and compression method).

ETC1 textures are supported in Android and benefit from GPU hardware decompression.



    没错，它居然是Ericsson公司发明的压缩专利，好在是完全公开的，并被OpenGL ES规范所支持。它基于这样一个事实：人类的视觉系统对明度luminance的敏感度高于色度chrominance。运用这一逆天的道理就能得到逆天的压缩方式，如下图：







1、先对图片进行分块4*4；

2、每块取出2个base colors（上下各一个，或者左右各一个），形成左图；

3、每块取出明度数据（逐像素的，那就是4*4=16个）；

4、算法合成得到右边的压缩后的效果；

　　所以，ETC1要做的就是把上面2和3中产生的base colors数据和luminance数据给整起来，一起压缩咯！这真是蛋疼，用2个基色块就能代表16个像素的颜色，你当玩家眼瞎啊！？

　　原理不多说了，直接上图：







1、4*4的像素块分为左右或者上下两个部分，提取2个base colors，采用差分存储，一个存为R5G5B5，差值存为dR3dG3dB3；

2、如果2个base colors相差太大，导致差值溢出，则直接存2个R4G4B4，比如左边红右边绿的这种极端情况;

3、两个标记位，绿色的那位表示是555差分存储呢，还是444独立存储；

4、重点是这个table bits，代表这里两个base color对应的是明度表里的index，正好3个bits，表里也一共2^3=8项；

5、位置刚好够用：16个RGB像素，编码进4个bytes，平均下来每个像素只需2个bits；

　　Texture被GPU加载到显存缓冲区之后，怎么寻址，怎么采样，具体细节就不深究了，那是硬件工程师的事情。总之，在这种框架下，ETC的缺点就是：

a、不支持RGBA 4通道的图片压缩；

b、对颜色连续过渡变化的图片压缩后可能有点儿糊（所以美术同学自带的像素眼会说你把他的图搞糊了）；

　　好了，上面就是对项目中常见的材质压缩问题的终极总结，有了这些知识点，回答前头的日常问题应该心中有数了（结合项目具体情况取舍）。

 

参考资料：

https://software.intel.com/en-us/articles/accelerating-texture-compression-with-intel-streaming-simd-extensions

http://fileadmin.cs.lth.se/cs/Education/EDAN35/lectures/L8-texcomp.pdf

http://www.powervr.com/Downloads/Factsheets/PVRTextureCompression.pdf

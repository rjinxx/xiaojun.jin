---
title: 与OpenGL ES的第一次约会
subtitle: ""

# Summary for listings and search engines
summary: 最近项目中需要实现一个实时视频绘制的功能，在相机中根据识别到的人脸点位信息，对指定的点之间绘制出图案来引导用户。出于性能的考量，决定采用OpenGL ES来进行图案的绘制。

# Link this post with a project
projects: []

# Date published
date: "2019-10-08T00:00:00Z"

# Date updated
lastmod: "2019-10-08T00:00:00Z"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: ''
  focal_point: ""
  placement: 2
  preview_only: false

authors:
- 金小俊

tags:
- OpenGL ES

categories:
- OpenGL ES
---

最近项目中需要实现一个实时视频绘制的功能，在相机中根据识别到的人脸点位信息，对指定的点之间绘制出图案来引导用户。出于性能的考量，决定采用OpenGL ES来进行图案的绘制。最终效果如下图所示：

![image](http://upload-images.jianshu.io/upload_images/268805-c0dfdb4416595120?imageMogr2/auto-orient/strip)

本文将从OpenGL的基础理论开始，由浅入深，直至实现上图的绘制效果。任何理论都不如现实具体，所以要想真正了解一门技术，必须从实际项目应用中去学习和实践。好了，我们开始吧！

### OpenGL ES

OpenGL(Open Graphics Library)是指定义了一个跨编程语言、跨平台的编程接口规格的专业的图形程序接口。其主要用于三维图像的绘制（当然，二维也可以），是一个功能强大，调用方便的底层图形库。而OpenGL ES则是OpenGL针对移动端的轻量级版本，简化了部分方法和数据类型，比如所有的图形都是由点、线和三角形组成。

我们知道在iOS中有两套常用的绘图框架。如下图所示，分别是UIKit和Core Graphics. 其中UIKit主要是用UIBezierPath来实现图形的绘制，实际上UIBezierPath是对Core Graphics框架的进一步封装。而Core Graphics则是使用Quartz2D做引擎，并且和OpenGL ES一样，在GPU上进行图形的绘制和渲染。

![image](http://upload-images.jianshu.io/upload_images/268805-ab1583d2c8e1343a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

那么问题来了，既然有这么多图形绘制框架，为什么要使用OpenGL呢？在计算机系统中CPU和GPU是协同工作的，CPU准备好显示数据后提交到GPU进行渲染，GPU渲染后将结果放入帧缓冲区，再经过数模转换最终由显示器显示出图像内容。由此可见，尽可能让CPU和GPU各司其职发挥作用是提高渲染效率的关键。
而OpenGL则让我们能够直接访问GPU，并且引入了缓存的概念来提升图形渲染的效率。

### 坐标系

首先我们来看下OpenGL的坐标系，如下图所示，以屏幕中心原点，坐标范围为-1到1之间。而我们平常接触的UIKit的坐标则是以屏幕左上角为原点，坐标范围则为屏幕宽高。

![image](http://upload-images.jianshu.io/upload_images/268805-db245b8cb606dba1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以如果我们在屏幕上通过OpenGL绘制图案就需要将UIKit的坐标系转换到OpenGL坐标系（这里主要讨论2D绘图，因此我们暂时忽略OpenGL的z轴），坐标转换的公式应该不难总结出来:

![image](http://upload-images.jianshu.io/upload_images/268805-4e726cb50dd031b1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 绘制流程

OpenGL ES 2.0的渲染流程如图所示，其中需要我们控制的为Vertex Data，Vertex Shader和Fragment Shader这三步。Vertex Data就是我们传入的顶点绘制数据，这里的顶点可以是表征点，线或者三角形的数据。Vertex Shader和Fragment Shader这两步是可编程的，也就是我们在下面将要见到的`.glsl`文件。Vertex Shader负责处理每一个点的顶点数据，而Fragment Shader则是针对像素数据的，其负责处理每个像素数据。

![image](http://upload-images.jianshu.io/upload_images/268805-80b40d71c6de11c8?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在OpenGL中，除非加载有效的顶点（Vertex Shader）和片段（Fragment Shader）着色器，否则不会绘制任何几何图形。我们先来看一个最基本的顶点着色器：

```c
// vertex.glsl
attribute vec4 position; 
void main(void) {
    gl_Position = position; 
}
```
第一行声明了一个名为position的4分量向量，并在`main`函数里面赋值给`gl_Position`变量。这里的`gl_Position`就是代表我们需要处理的顶点，也就是上图中的Vertex Data数据。

> 在shader中一共有三种变量类型`attribute`, `uniform`和`varying`. 其区别为:`uniform`变量是外部程序传递给shader的变量；`attribute`变量只能在vertex shader中使用，为外部程序传递给vertex shader的变量；`varying`变量则是vertex和fragment shader之间做数据传递用的。

我们接着再来看片段着色器的一段代码：

```c
// fragment.glsl
precision mediump float;
void main(void) {
    gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0); 
}
```

第一行是声明着色器中浮点变量的默认精度。接着在`main`函数里面赋值每个像素的颜色值，这里我们赋值`vec4(1.0, 0.0, 0.0, 1.0)`代表每个像素点的颜色都是红色。

### 基本图元

使用OpenGL绘制图形一般都是从绘制一个三角形开始，因为这个过程包括了OpenGL ES的三种基本元素: 点，线和三角。在OpenGL中，任何复杂的三维模型都是由这三个基本的几何图元组成的。

#### 编译着色器

顶点和像素的处理都是在shader中实现的，所以我们要想使用shader就需要在运行时动态编译源码以得到一个着色器对象。幸运的是，编译shader的流程是固定的，而且已经有很多现成的开源代码实现。其大概步骤如下所示：

首先是编译shader的代码，其中`path`为`vertex.glsl`或者`vertex.glsl`文件的存放路径，而`type`则是用来区分shader的种类，即Vertex Shader或者Fragment Shader着色器。

```objc
- (GLuint)compileShader:(NSString *)path type:(GLenum)type source:(GLchar *)source
{
    NSError *error          = nil;
    NSString *shaderContent = [NSString stringWithContentsOfFile:path encoding:NSUTF8StringEncoding error:&error];
    
    if (!shaderContent) NSLog(@"%@", error.localizedDescription);
    
    const char *shaderUTF8 = [shaderContent UTF8String];
    GLint length           = (GLint)[shaderContent length];
    GLuint shader          = glCreateShader(type);
    
    glShaderSource(shader, 1, &shaderUTF8, &length);
    
    glCompileShader(shader);
    
    GLint status;
    glGetShaderiv(shader, GL_COMPILE_STATUS, &status);
    
    if (status == GL_FALSE) { glDeleteShader(shader); exit(1); }
    
    return shader;
}
```

现在我们有了编译之后的shader对象，接下来需要把它链接到OpenGL的`glProgram`上，让它可以在GPU上run起来。代码如下所示：

```
program = glCreateProgram();

glAttachShader(program, vertShader);
glAttachShader(program, fragShader);

glLinkProgram(program);
    
GLint status;
glGetProgramiv(program, GL_LINK_STATUS, &status);
```

完成上面的步骤后，我们就可以用`programe`来和shader交互了，比如赋值给顶点shader的`position`变量：

```
GLuint attrib_position = glGetAttribLocation(program, "position");
glEnableVertexAttribArray(attrib_position);
glVertexAttribPointer(attrib_position, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (char *)points);
```

#### 几何图元

有了上面的介绍，我们就可以开始绘图了。所有几何图元的绘制都是通过调用`glDrawArrays`实现的：
```objc
glDrawArrays (GLenum mode, GLint first, GLsizei count);
```
这里的`mode`为几何形状类型，主要有点，线和三角形三种：

```objc
#define GL_POINTS           0x0000  // 点     -> 默认为方形
#define GL_LINES            0x0001  // 线段   -> 可不连续
#define GL_LINE_LOOP        0x0002  // 线圈   -> 首尾相连的线段
#define GL_LINE_STRIP       0x0003  // 线段带 -> 相邻线段共享顶点
#define GL_TRIANGLES        0x0004  // 三角形 -> 三个顶点连接
#define GL_TRIANGLE_STRIP   0x0005  // 三角带 -> 相邻三角共享边
#define GL_TRIANGLE_FAN     0x0006  // 三角扇 -> 所有三角共享顶点
```

绘制点代码如下所示，其中几何类型传入`GL_POINTS`

```objc
static GLfloat points[] = { // 前三位表示位置x, y, z 后三位表示颜色值r, g, b                 
    0.0f, 0.5f, 0, 0, 0, 0, // 位置为( 0.0, 0.5, 0.0); 颜色为(0, 0, 0)黑色
   -0.5f, 0.0f, 0, 1, 0, 0, // 位置为(-0.5, 0.0, 0.0); 颜色为(1, 0, 0)红色       
    0.5f, 0.0f, 0, 1, 0, 0  // 位置为( 0.5, 0.0, 0.0); 颜色为(1, 0, 0)红色       
}; // 共有三组数据，表示三个点

GLuint attrib_position = glGetAttribLocation(program, "position");
glEnableVertexAttribArray(attrib_position);
GLuint attrib_color    = glGetAttribLocation(program, "color");
glEnableVertexAttribArray(attrib_color);

// 对于position每个数值包含3个分量，即3个byte，两组数据间间隔6个GLfloat
// 同样,对于color每个数值含3个分量，但数据开始的指针位置为跳过3个position的GLFloat大小
glVertexAttribPointer(attrib_position, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (char *)points);
glVertexAttribPointer(attrib_color, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (char *)points + 3 * sizeof(GLfloat));
 
glDrawArrays(GL_POINTS, 0, 3); 
```

效果如图所示：

![image](http://upload-images.jianshu.io/upload_images/268805-5bb299b56fe34eef?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到绘制出来的点默认为方点，那如果要绘制圆点呢？为了让OpenGL ES 2.0把点绘制成圆形而非矩形，需要处理光栅化后的点所包含的像素数据，思路是，忽略半径大于0.5的点，从而实现圆点绘制。在`FragmentShader.glsl`修改代码如下：

```c
// FragmentShader.glsl
varying lowp vec4 fragColor;

void main(void) {
    if (length(gl_PointCoord - vec2(0.5, 0.5)) > 0.5) {
        discard;
    }
    gl_FragColor = fragColor;
}
```

运行后，可以看到圆点效果如下所示：

![image](http://upload-images.jianshu.io/upload_images/268805-5a014c122030609a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
    
绘制直线的代码如下所示，其中几何类型传入`GL_LINES`

```objc
static GLfloat lines[] = { 
    0.0f, 0.0f, 1, 1, 1, 1,
    0.5f, 0.5f, 0, 0, 0, 0,
    0.0f, 0.0f, 0, 1, 0, 0,
   -0.5f, 0.0f, 0, 0, 0, 1,
};

glVertexAttribPointer(attrib_position, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (char *)lines);
glVertexAttribPointer(attrib_color, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(GLfloat), (char *)lines + 3 * sizeof(GLfloat));
 
glLineWidth(5); // 设置线宽为5
glDrawArrays(GL_LINES, 0, 4); 
```

对于线段，如果两点之间的颜色值不同，则OpenGL会默认产生渐变色效果，具体绘制结果如图所示：

![image](http://upload-images.jianshu.io/upload_images/268805-fe839bdc4e196647?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由于本文最开始的效果里面只用到了点和线的绘制，所以绘制最基本的三角形，读者可以自行尝试，这边就不再赘述了。

### 纹理贴图

除了图元之外，OpenGL还有纹理的概念。简单来说就是把图像数据显示到我们所绘制的图元上，以使图元表示的物体更真实。我们首先来看下纹理的坐标系，如下图所示：

![image](http://upload-images.jianshu.io/upload_images/268805-f6e4f98f5cefd836?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

纹理坐标的范围为0到1之间。纹理坐标的原点为图片的左下角，其和OpenGL绘制坐标系的对应关系如示意图上箭头所示，在纹理贴图的时候我们需要确保坐标点映射关系与上图一致。

要实现纹理的绘制需要两个信息，一个是纹理的坐标，另一个则是纹理的内容。纹理的内容简单来说，就是把iOS中的UIImage转换为OpenGL ES中的texture数据。

```objc
- (GLuint)textureFromImage:(UIImage *)image 
{
    CGImageRef imageRef = [image CGImage];
    size_t w = CGImageGetWidth (imageRef);
    size_t h = CGImageGetHeight(imageRef);
    
    GLubyte *textureData        = (GLubyte *)malloc(w * h * 4);
    CGColorSpaceRef colorSpace  = CGColorSpaceCreateDeviceRGB();
    
    NSUInteger bytesPerPixel    = 4;
    NSUInteger bytesPerRow      = bytesPerPixel * w;
    NSUInteger bitsPerComponent = 8;
    
    CGContextRef context = CGBitmapContextCreate(textureData,
                                                 w,
                                                 h,
                                                 bitsPerComponent, 
                                                 bytesPerRow, 
                                                 colorSpace,
                                                 kCGImageAlphaPremultipliedLast | kCGBitmapByteOrder32Big);
    CGContextTranslateCTM(context, 0, h);
    CGContextScaleCTM(context, 1.0f, -1.0f);
    CGContextDrawImage(context, CGRectMake(0, 0, w, h), imageRef);
    
    glEnable(GL_TEXTURE_2D);
    GLuint texName;
    glGenTextures(1, &texName);
    glBindTexture(GL_TEXTURE_2D, texName);
    
    glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    
    glTexImage2D(GL_TEXTURE_2D, 
                 0, 
                 GL_RGBA, 
                 (GLsizei)w, 
                 (GLsizei)h, 
                 0,
                 GL_RGBA, 
                 GL_UNSIGNED_BYTE, 
                 textureData);
    
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    free(textureData);
    
    return texName;
}
```

有了纹理对象后，接下来我们需要在顶点着色器和片段着色器中转化坐标和纹理信息，也就是进行采样渲染。顶点着色器如下所示：

```c
// vertex.glsl
attribute vec4 aPosition; 
attribute vec2 aTexcoord;
varying   vec2 vTexcoord;
void main(void) {
    gl_Position = aPosition; 
    vTexcoord   = aTexcoord;
}
```

上述代码中的`aTexcoord`用来接受纹理坐标信息，然后传递给片段着色器中定义的varying变量vTexcoord。这样就传递了纹理坐标信息。片段着色器代码如下所示：

```c
// fragment.glsl
precision mediump   float;
uniform   sampler2D uTexture;
varying   vec2      vTexcoord;
void main(void) {
    gl_FragColor = texture2D(uTexture, vTexcoord);
}
```

这里的`uTexture`就是我们的纹理，而`vTexcoord`则是纹理坐标。有了坐标和纹理信息后就可以通过`texture2D`函数进行采样。简单来说，就是取出每个坐标点像素的颜色信息赋给OpenGL进行绘制，而图片的数据就是由每个点的颜色像素值所组成的矩阵信息，因此，有了纹理和像素间的颜色映射关系后，就可以通过OpenGL显示整张图片了。完成了上述操作之后，最后一步就是激活纹理并渲染了，代码如下所示：

```objc
GLuint tex_name = [self textureFromImage:[UIImage imageNamed:@"ryan.jpg"]];

glActiveTexture(GL_TEXTURE5);
glBindTexture(GL_TEXTURE_2D, tex_name);
glUniform1i(uTexture, 5);

const GLfloat vertices[] = { // OpenGL绘制坐标
    -0.5, -0.25, 0,   
     0.5, -0.25, 0,   
    -0.5,  0.25, 0,   
     0.5,  0.25, 0 }; 
glEnableVertexAttribArray(aPosition);
glVertexAttribPointer(aPosition, 3, GL_FLOAT, GL_FALSE, 0, vertices);

static const GLfloat coords[] = { // 纹理坐标
    0, 0,
    1, 0,
    0, 1,
    1, 1
};

glEnableVertexAttribArray(aTexcoord);
glVertexAttribPointer(aTexcoord, 2, GL_FLOAT, GL_FALSE, 0, coords);

glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
```

代码中的`vertices`为OpenGL的绘制坐标，纹理坐标为`coords`, 这两个坐标需要与上图的坐标对应关系相符合才能正确显示出图片。运行后效果如下图所示：

![image](http://upload-images.jianshu.io/upload_images/268805-d6ea09508516151e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 视频绘制

好了，有了上面的理论基础，我们可以来实现文章开篇所示的实时视频绘制了。对于视频流的获取以及OpenGL的绘制环境我们采用GPUImage来实现，人脸识别的算法采用公司自有视觉引擎（免费开放使用，下载地址为[虹软视觉AI引擎开放平台](https://ai.arcsoft.com.cn/index.html)）当然也可以使用CoreImage框架的CIDetector人脸识别类。

```
@interface PVTStickerFilter : GPUImageFilter

@property (nonatomic, copy) NSArray<NSValue *> *facePoints;

@end
```

首先继承`GPUImageFilter`类，并定义一个人脸点位数组用来接收人脸识别引擎传入的点位信息。需要注意的是，相机获取的图像默认在内存中是逆时针90度存放的，所以我们获取的点位需要顺时针旋转90度才是我们在取景框中看到的图像。另外，如果是前置摄像头，默认会有镜像效果，因此还需要将点位沿Y轴翻转180度。

```objc
[self.facePoints enumerateObjectsUsingBlock:^(NSValue *obj, NSUInteger idx, BOOL *stop) {
    CGPoint point = [obj CGPointValue];
    [mPs addObject:[NSValue valueWithCGPoint:CGPointMake(point.y, point.x)]];
}];
```

> 对于某个点`(x, y)`顺时针旋转90度后坐标为`(imageHeight - y, x)`, 如果是镜像效果的点，则还需要再绕Y轴旋转180度，最终的坐标为`(y, x)`。

从效果图中可以看到，我们要实现的为左右两边对称线条的动画绘制。效果图中一共绘制了三组线条，我们就其中一组来分析下其原理。具体点位为鼻梁左下角点`(x67, y67)`到眉毛左内侧点`(x24, y24)`的线段绘制，以及鼻梁右下角点`(x70, y70)`到眉毛右内侧点`(x29, y29)`的线段绘制。同时`(x24, y24)`和`(x29, y29)`在动画的最后还需要显示圆点。

根据前文的分析，在绘制点位之前我们还需要把视频图像帧的坐标转换为OpenGL的坐标系，也就是把上面几个点位的坐标转换到-1到1之间。转换公式前文已给出：

```objc
CGFloat x67 = 2 * [mPs[67] CGPointValue].x / frameWidth - 1.f;
CGFloat y67 = 1 - 2 * [mPs[67] CGPointValue].y / frameHeight ;

CGFloat x24 = 2 * [mPs[24] CGPointValue].x / frameWidth - 1.f;
CGFloat y24 = 1 - 2 * [mPs[24] CGPointValue].y / frameHeight ;

CGFloat x70 = 2 * [mPs[70] CGPointValue].x / frameWidth - 1.f;
CGFloat y70 = 1 - 2 * [mPs[70] CGPointValue].y / frameHeight ;

CGFloat x29 = 2 * [mPs[29] CGPointValue].x / frameWidth - 1.f;
CGFloat y29 = 1 - 2 * [mPs[29] CGPointValue].y / frameHeight ;
```

有了这些点位，我们可以很容易的使用`glDrawArrays(GL_LINES, 0, 4)`来绘制出线段。但是这边有两个问题需要解决，一是如何绘制虚线，二是如何实现绘制的动画。 

对于虚线的绘制，OpenGL ES 2.0没有直接的API可以实现，所以我们需要换一种思路，将虚线转换为若干直线的连续绘制。具体思路为，一个长度为10像素的虚线`(x1, 0)`至`(x10, 0)`，我们将它切断为5个长度为1像素线段绘制。即绘制`(x1, 0)`到`(x2, 0)`的线段，`(x3, 0)`到`(x4, 0)`的线段，`(x5, 0)`到`(x6, 0)`的线段，`(x7, 0)`到`(x8, 0)`的线段，`(x9, 0)`到`(x10, 0)`的线段。

所以，首先我们需要根据绘制虚线的长度来给整条线段分段，比如我们定义每段虚线的长度为0.01，那么就可以计算出来两个点位之间的线段需要分为多少片段线来绘制：

```objc
CGFloat w_24_67 = (x24 - x67); // 两点之间的x轴距离
CGFloat h_24_67 = (y24 - y67); // 两点之间的y轴距离

CGFloat w_29_70 = (x29 - x70); // 两点之间的x轴距离
CGFloat h_29_70 = (y29 - y70); // 两点之间的y轴距离

GLsizei s_24_67 = [self stepsOfLineWidth:w_24_67 height:h_24_67]; // 需要划分为多少个片段线
GLsizei s_29_70 = [self stepsOfLineWidth:w_29_70 height:h_29_70]; // 需要划分为多少个片段线
```

计算片段性的函数如下所示，其中`PVT_DASH_LENGTH`为每段虚线的长度：

```objc
- (GLsizei)stepsOfLineWidth:(CGFloat)w height:(CGFloat)h
{
    CGFloat a_w = fabs(w);
    CGFloat a_h = fabs(h);
    GLsizei s   = a_w / (PVT_DASH_LENGTH * cos(atan(a_h / a_w)));
    
    return ((s % 2) ? s : ++s) + 1;
}
```

然后将所有的线段片塞到OpenGL中绘制，代码如下：

```objc
GLsizei total_s = s_24_67 + s_29_70;
GLfloat *lines  = (GLfloat *)malloc(sizeof(GLfloat) * total_s * 3);

for (int i = 0; i < s_24_67; i++) {
    CGFloat xt = x67 + (CGFloat)i/(CGFloat)(s_24_67-1) * w_24_67;
    CGFloat yt = y67 + (CGFloat)i/(CGFloat)(s_24_67-1) * h_24_67;
    int   idx  = i * 3;
    lines[idx] = xt; lines[idx+1] = yt; lines[idx+2] = 0;
}
for (int i = 0; i < s_29_70; i++) {
    CGFloat xt = x70 + (CGFloat)i/(CGFloat)(s_29_70-1) * w_29_70;
    CGFloat yt = y70 + (CGFloat)i/(CGFloat)(s_29_70-1) * h_29_70;
    int   idx  = s_24_67 * 3 + i * 3;
    lines[idx] = xt; lines[idx+1] = yt; lines[idx+2] = 0;
}

glVertexAttribPointer(_position, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (char *)lines);
glLineWidth(2.5);
glDrawArrays(GL_LINES, 0, total_s);
```

好了，虚线的问题我们解决了，我们再来看看如何实现绘制的动画。其实思路很简单，比如我们要在4秒内逐步绘制出线段（由于需要绘制虚线，我们分成了100个线段片），那么，我们在相机每帧数据回调来的时候判断下当前帧距离第一帧已经间隔了多次时间，假设间隔了1秒，那就是对于这一帧图像我们需要绘制出四分之一的长度，也就是将25个线段片塞到OpenGL里面去绘制。以此类推，如果超过了4秒，那么再清零重头计算。在4秒的时候应该是绘制整条线段的完整长度。

```objc
- (void)newFrameReadyAtTime:(CMTime)frameTime atIndex:(NSInteger)textureIndex
{
    _currentTime = frameTime;

    [super newFrameReadyAtTime:frameTime atIndex:textureIndex];
}
```

首先记录下当前帧的时间，以便在后面计算当前帧距离第一帧的累积时间。

```objc
- (void)calcAccumulatorTime
{
    NSTimeInterval interval = 0;
    
    if (CMTIME_IS_VALID(_lastTime)) {
        interval = CMTimeGetSeconds(CMTimeSubtract(_currentTime, _lastTime));
    }
    _lastTime       = _currentTime;
    _accumulator   += interval;
    
    _frameDuration  = _stepsIdx == 3 ? PVT_FRAME_DURATION / 2.f : PVT_FRAME_DURATION;
    
    CGFloat sumTime = _accumulator + interval;
    _accumulator    = MIN(sumTime, _frameDuration);
}
```

然后计算出当前帧根据总的动画时间应该绘制到哪一步：

```objc
- (GLsizei)animationIdxWithStep:(GLsizei)step
{
    CGFloat s_scale = _accumulator / _frameDuration;
    GLsizei s_index = ceil(s_scale * step);
    
    return (s_index % 2) ? ++s_index : s_index;
}
```
最后一步则是将计算好的片段数传给OpenGL进行绘制，需要注意的时候当累积时间超过了动画时间后需要将累积时间清零，从而实现动画的连续展示。这里的`_frameDuration`即是动画时间。

```objc
- (void)renderToTextureWithVertices:(const GLfloat *)vertices textureCoordinates:(const GLfloat *)textureCoordinates;
{
    [self calcAccumulatorTime];

    GLsizei s_24_67_index = [self animationIdxWithStep:s_24_67];
    GLsizei s_29_70_index = [self animationIdxWithStep:s_29_70];

    GLsizei total_s = s_24_67_index + s_29_70_index;
    GLfloat *lines  = (GLfloat *)malloc(sizeof(GLfloat) * total_s * 3);
    
    for (int i = 0; i < s_24_67_index; i++) {
        CGFloat xt = x67 + (CGFloat)i/(CGFloat)(s_24_67_index-1) * w_24_67 * s_index_scale;
        CGFloat yt = y67 + (CGFloat)i/(CGFloat)(s_24_67_index-1) * h_24_67 * s_index_scale;
        int   idx  = i * 3;
        lines[idx] = xt; lines[idx+1] = yt; lines[idx+2] = 0;
    }
    for (int i = 0; i < s_29_70_index; i++) {
        CGFloat xt = x70 + (CGFloat)i/(CGFloat)(s_29_70_index-1) * w_29_70 * s_index_scale;
        CGFloat yt = y70 + (CGFloat)i/(CGFloat)(s_29_70_index-1) * h_29_70 * s_index_scale;
        int   idx  = s_24_67_index * 3 + i * 3;
        lines[idx] = xt; lines[idx+1] = yt; lines[idx+2] = 0;
    }
    
    if (_accumulator == _frameDuration) {
        _accumulator = 0.f;
    }
    
    // to do drawing work...
}
```

虚线和动画的问题都解决了，现在还剩最后一个需求，在动画结束的时候在(x24, y24)和(x29, y29)处绘制圆点。对于圆点的绘制，前文有提到可以直接绘制点，然后在FragmentShader.glsl中修改忽略半径大于0.5的即可实现圆点绘制。但是由于我们需要同时绘制点和线，且使用同一个Fragment Shader文件，所以难以区分当前是绘制点还是线，不能直接在Shader中忽略半径大于0.5的点，因此我们这边对于圆点直接采用几何方法绘制。具体的几何原理可以参照[这篇](https://www.jianshu.com/p/f929f83ad0d7)博文。

```objc
#define PVT_CIRCLE_SLICES  100
#define PVT_CIRCLE_RADIUS  0.015

- (void)drawCircleWithPositionX:(CGFloat)x y:(CGFloat)y radio:(CGFloat)radio
{
    glLineWidth(2.0);
    
    GLfloat *vertext = (GLfloat *)malloc(sizeof(GLfloat) * PVT_CIRCLE_SLICES * 3);
    
    memset(vertext, 0x00, sizeof(GLfloat) * PVT_CIRCLE_SLICES * 3);
    
    float a     = PVT_CIRCLE_RADIUS; // horizontal radius
    float b     = a * radio;         // fWidth / fHeight;
    
    float delta = 2.0 * M_PI / PVT_CIRCLE_SLICES;
    
    for (int i = 0; i < PVT_CIRCLE_SLICES; i++) {
        GLfloat cx   = a * cos(delta * i) + x;
        GLfloat cy   = b * sin(delta * i) + y;
        int   idx    = i * 3;
        vertext[idx] = cx; vertext[idx+1] = cy; vertext[idx+2] = 0;
    }
    
    glVertexAttribPointer(_position, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (char *)vertext);
    glDrawArrays(GL_TRIANGLE_FAN, 0, PVT_CIRCLE_SLICES);
    
    free(vertext);
}
```

OpenGL ES的深度不亚于学习一门新语言，万丈高楼平地起，希望本文的总结可以给想入门的同学带来一些帮助和收获，也欢迎大家留言讨论。

### 参考文章
1. [OpenGL ES入门及绘制一个三角形](http://linbinghe.com/2018/b8f62c8f.html)
2. [仿QQ视屏动画特效－人脸识别](https://blog.csdn.net/hbw1992322/article/details/56272838)
3. [从0打造一个GPUImage](https://zhuanlan.zhihu.com/p/29795080)
4. [学习OpenGL ES之绘制更多的图形](https://www.jianshu.com/p/c269dac0f919)
5. [OpenGL ES 3.0 数据可视化 1：绘制圆点](https://www.jianshu.com/p/80dff12b57b7)
6. [OpenGL ES入门03-OpenGL ES圆形绘制](https://www.jianshu.com/p/f929f83ad0d7)
7. [OpenGL ES入门05-OpenGL ES 纹理贴图](https://www.jianshu.com/p/4d8d35288a0f)
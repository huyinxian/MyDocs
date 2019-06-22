# 矩阵变换与Unity空间坐标转换

这一篇笔记将对矩阵的应用进行拓展，同时也会介绍一下 Unity 中的各类空间转换。

## 几种常见的矩阵变换

---

在三维渲染中，矩阵可以视为是一种变换，这些变化一般包含了旋转、缩放、平移等。开发人员希望给定一个点或者向量，再给定一个变换（比如把点平移，或者把向量旋转一下），就可以通过某种数学运算得到新的点和向量。那么矩阵就是用来解决这个问题的。

### 什么是变换

所谓的变换，是指我们将一些数据，比如点、矢量、颜色等通过某种方式进行转换的过程。我们先来看一下**线性变换**，它满足如下两个条件：

$$ f(x)+f(y)=f(x+y) $$
$$ kf(x)=f(kx) $$

举个例子，缩放就是一个线性变换。我们假设 $f(x)=2\boldsymbol x$，那么这就表示一个大小为 2 的统一缩放，即经过变换后向量 $\boldsymbol x$ 的模将放大两倍。同样的，旋转也是一种线性变换。对于线性变换来说，如果要对一个三维向量进行变换，那么只需要使用 3X3 的矩阵就能够表示所有的线性变换。除了上述的两种，线性变换还有很多，不过用的最多的还是旋转和缩放变换。

不过呢，仅有线性变换是不够的，我们还需要考虑平移变换，比如 $f(x)=\boldsymbol x + (1,2,3)$。但问题是，平移变换并不满足线性变换的两个条件，因而也无法用 3X3 的矩阵来表示平移变换。这是我们不希望看到的，因为平移变换也是很常见的一种变换。

基于这种需求，**仿射变换**出现了。仿射变换是线性变换和平移变换的集合，它使用一个 4X4 矩阵来表示。为此，我们需要把向量拓展到四维空间，也就是**齐次坐标空间**。

### 齐次坐标

当我们将三维向量扩展成四维向量时，就成为了**齐次坐标**。齐次坐标可以继续往下拓展，但这不在我们的讨论范围内。

既然齐次坐标是一个四维向量，那么第四个值 $w$ 是怎么来的呢？我们可以先想象一个位于 $w=1$ 平面的 2D 坐标 $(x, y, 1)$。对于那些不处于 $w=1$ 平面的坐标，我们将它们全部都投影到该平面上，所以齐次坐标 $(x, y, w)$ 映射出来的 2D 坐标为 $(x/w, y/w)$。因此，对于一个 2D 坐标而言，在齐次空间中存在无数个与之对应的点，它们的坐标可以表示为 $(kx,ky,k)$（k≠0）。

同样的，3D 坐标也是位于 $w=1$ 平面上，所以对于四维齐次坐标 $(x,y,z,w)$ 而言，它所对应的 3D 坐标为 $(x/w,y/w,z/w)$。我们把这种将齐次空间内的点投影到 $w=1$ 平面上的方式称之为**齐次除法**，该计算方式常用于投影矩阵的计算。

当 $w=0$ 时，由于无法使用除法，因此该坐标点相当于一个无穷远处的点，它描述的是方向而不是位置。

### 分解基础变换矩阵

4X4 的矩阵可以用来表示平移、旋转、缩放变换。我们把表示纯平移、纯旋转、纯缩放的变换矩阵叫做基础变换矩阵。这种矩阵可以像下面这样分解：

$$
\begin{bmatrix}
   \boldsymbol M_{3 \times 3} & \boldsymbol t_{3 \times 1} \\
   \boldsymbol 0_{1 \times 3} & 1
\end{bmatrix}
$$

左上角的矩阵 $\boldsymbol M_{3 \times 3}$ 表示旋转和缩放，$\boldsymbol t_{3 \times 1}$ 用于表示平移，$\boldsymbol 0_{1 \times 3}$ 是零矩阵，右下角是标量 1。

### 平移矩阵

下面我们对一个点进行平移变换：

$$
\begin{bmatrix}
   1 & 0 & 0 & t_{x} \\
   0 & 1 & 0 & t_{y} \\
   0 & 0 & 1 & t_{z} \\
   0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
   x \\
   y \\
   z \\
   1
\end{bmatrix} =
\begin{bmatrix}
   x + t_{x} \\
   y + t_{y} \\
   z + t_{z} \\
   1
\end{bmatrix}
$$

从结果可以看出，点 $(x,y,z)$ 平移了 $(t_{x},t_{y},t_{z})$ 个单位。我们还可以顺便看下对向量进行平移变换的结果：

$$
\begin{bmatrix}
   1 & 0 & 0 & t_{x} \\
   0 & 1 & 0 & t_{y} \\
   0 & 0 & 1 & t_{z} \\
   0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
   x \\
   y \\
   z \\
   0
\end{bmatrix} =
\begin{bmatrix}
   x \\
   y \\
   z \\
   0
\end{bmatrix}
$$

可以看出，由于向量的第四个分量为 0，因此平移变换对于向量是不起作用的（向量没有位置概念）。

### 缩放矩阵

缩放变换如下：

$$
\begin{bmatrix}
   k_{x} & 0 & 0 & 0 \\
   0 & k_{y} & 0 & 0 \\
   0 & 0 & k_{z} & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
   x \\
   y \\
   z \\
   1
\end{bmatrix} =
\begin{bmatrix}
   k_{x}x \\
   k_{y}y \\
   k_{z}z \\
   1
\end{bmatrix}
$$

如果缩放系数 $k_{x}=k_{y}=k_{z}$，那么这种缩放就称为**统一缩放**。统一缩放的效果是将模型统一扩大或缩小，不会让模型失去原有比例。

缩放矩阵的逆矩阵如下：

$$
\begin{bmatrix}
   \frac{1}{k_{x}} & 0 & 0 & 0 \\
   0 & \frac{1}{k_{y}} & 0 & 0 \\
   0 & 0 & \frac{1}{k_{z}} & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}$$

?> 上述的缩放矩阵只能沿坐标轴方向缩放，如果要向任意方向缩放需要使用复合变换。

### 旋转矩阵

旋转矩阵比较麻烦，因为旋转的时候需要制定一个旋转轴，并且这个旋转轴还不一定就是坐标轴。当然，目前来说我们的旋转轴默认为 x、y、z 轴。

首先是绕着 x 轴旋转 $\theta$ 角度：

$$
\begin{bmatrix}
   1 & 0 & 0 & 0 \\
   0 & \cos\theta & -\sin\theta & 0 \\
   0 & \sin\theta & \cos\theta & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}$$

然后是 y 轴：

$$
\begin{bmatrix}
   \cos\theta & 0 & \sin\theta & 0 \\
   0 & 1 & 0 & 0 \\
   -\sin\theta & 0 & \cos\theta & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}$$

最后是 z 轴：

$$
\begin{bmatrix}
   \cos\theta & -\sin\theta & 0 & 0 \\
   \sin\theta & \cos\theta & 0 & 0 \\
   0 & 0 & 1 & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}$$

如果你不希望死记硬背，那么你可以换一种思路进行理解。首先，缩放和旋转是用不到四阶矩阵的，可以简化为三阶。接下来再将每一行视作是单独的基向量：

$$
\begin{bmatrix}
   \cos\theta & -\sin\theta & 0 \\
   \sin\theta & \cos\theta & 0 \\
   0 & 0 & 1 
\end{bmatrix} = 
\begin{bmatrix}
   \boldsymbol p \\
   \boldsymbol q \\
   \boldsymbol l
\end{bmatrix}
$$

由于我们当前是围绕 z 轴进行旋转，因此基向量 l 是不变的。同样的，对于基向量 p、q 而言，它们的 z 坐标也是不变的，因此矩阵可以简化为二阶矩阵：

$$
\begin{bmatrix}
   \cos\theta & -\sin\theta \\
   \sin\theta & \cos\theta
\end{bmatrix}
$$

看到这里，想必你就会很熟悉了。同样的，对于绕 x、y 轴的旋转矩阵也能够进行对应的简化。

?> 旋转矩阵的逆矩阵就是旋转相反角度的变换矩阵。

### 复合变换

复合变换就是把上述的三种变换综合起来，形成一个复杂的变换过程。例如，对一个模型进行缩放、旋转、平移，可以用以下公式计算：

$$ \boldsymbol p_{new} = \boldsymbol M_{translation} \boldsymbol M_{rotation} \boldsymbol M_{scale} \boldsymbol p_{old} $$

我们使用的是列矩阵，因此阅读顺序是从右到左，也就是先缩放再旋转最后平移。由于矩阵乘法不满足交换律，因此顺序很重要，绝大多数情况下变换的顺序是先缩放再旋转最后平移。

为什么这样呢？想象一下我们是按照先平移再缩放的顺序进行变换，假设模型处于原点，让它先平移 (0,0,5)，然后放大两倍，那么这样一来所有坐标就会变为原来的两倍，也就是说模型的位置处于 (0,0,10)。显然，这样是错误的，我们应该先让模型放大，然后再进行平移，这样的话模型的位置就不会出错。

除了注意变换的顺序以外，我们还要注意旋转的先后顺序。如果一个物体同时绕着 3 个轴进行旋转，那么旋转的顺序应该是怎么样的呢？

当我们直接给出 $(\theta_{x},\theta_{y},\theta_{z})$ 这样的旋转角度时，我们需要规定一个旋转顺序。Unity 的旋转顺序是 $zxy$，这个可以在官方文档里看到：

$$
\boldsymbol M_{rotatez} \boldsymbol M_{rotatex} \boldsymbol M_{rotatey} = 
\begin{bmatrix}
   \cos\theta_{z} & -\sin\theta_{z} & 0 & 0 \\
   \sin\theta_{z} & \cos\theta_{z} & 0 & 0 \\
   0 & 0 & 1 & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
   1 & 0 & 0 & 0 \\
   0 & \cos\theta_{x} & -\sin\theta_{x} & 0 \\
   0 & \sin\theta_{x} & \cos\theta_{x} & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}
\begin{bmatrix}
   \cos\theta_{y} & 0 & \sin\theta_{y} & 0 \\
   0 & 1 & 0 & 0 \\
   -\sin\theta_{y} & 0 & \cos\theta_{y} & 0 \\
   0 & 0 & 0 & 1
\end{bmatrix}
$$

有人可能会问，上面的顺序是不是反了，不是说列矩阵都要从右往左读吗？是这样的，我们在确定了旋转顺序后，有两种坐标系可以旋转。第一种是在旋转模型的时候不对当前坐标系进行旋转，第二种是在旋转模型的时候带着当前坐标系一起旋转。显然，这两种做法的结果是不一样的，但如果把它们的旋转顺序颠倒一下，那么这种两种做法就能得到相同结果。

说的详细点，在第一种情况下按照 $zxy$ 旋转与在第二种情况下按照 $yxz$ 旋转得到的结果是一样的。Unity 文档中给出的旋转顺序是针对第一种情况而言的。

### 正交投影矩阵

正交投影是指将向量投影至某个轴或者某个平面上，其形式类似于现实中的太阳光所产生的影子。在 Unity 中，正交投影会显示出 2D 画面的效果，相当于将一个三维向量赋值给二维向量（即忽略掉 z 坐标）。

向 xy 平面的投影矩阵如下：

$$
\begin{bmatrix}
   1 & 0 & 0 \\
   0 & 1 & 0 \\
   0 & 0 & 0 
\end{bmatrix}
$$

向 xz 平面的投影矩阵如下：

$$
\begin{bmatrix}
   1 & 0 & 0 \\
   0 & 0 & 0 \\
   0 & 0 & 1 
\end{bmatrix}
$$

向 yz 平面的投影矩阵如下：

$$
\begin{bmatrix}
   0 & 0 & 0 \\
   0 & 1 & 0 \\
   0 & 0 & 1 
\end{bmatrix}
$$

### 透视投影矩阵

如果说正交投影相当于让三维坐标抛弃 z 坐标直接降为二维，那么透视投影则相当于让 x、y 坐标除以一个缩放因子。如果你有查看过 Unity 或者其他游戏引擎的相机视角，你就能发现透视投影所成的视野范围类似于一个视锥体，而视锥体中的点需要按照一定的缩放比例投影到对应的平面上，如下图所示。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/%E9%80%8F%E8%A7%86%E6%8A%95%E5%BD%B1%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

我们可以让相机原点 O 与物体顶点 B 相连，则直线 OB 与投影平面相交的点 A 就是顶点经过透视投影得到的坐标点。由于相机原点到两个平面的距离 OE、OF 是已知的，所以我们可以通过相似三角形 OEA 与 OFB 得到投影点 A 的坐标。

现在，假设 O 为原点，投影平面处在 `z=d` 的位置，则点 B 通过透视投影得到的结果如下：

$$
\boldsymbol p =
\begin{bmatrix}
   x \\
   y \\
   z 
\end{bmatrix} \Rightarrow
\boldsymbol p'
\begin{bmatrix}
   x' \\
   y' \\
   z' 
\end{bmatrix} =
\begin{bmatrix}
   dx/z \\
   dy/z \\
   d 
\end{bmatrix}
$$

当然，光是这样还不行，我们得想办法把上述结论转变成一种矩阵，否则透视投影将无法和其它的变换联系起来。在前面的内容中我们已经得知，如果要把一个 4D 的齐次向量变换到 3D 中，需要用 4D 向量除以 $w$，也就是说我们可以得到向量 $\boldsymbol p'$ 的表示形式：

$$
\begin{bmatrix}
   x & y & z & z/d
\end{bmatrix}
$$

从结果进行反推，对于一个 4D 齐次向量 $ (x,y,z,1) $，它的透视投影矩阵如下：

$$
\begin{bmatrix}
   x & y & z & 1
\end{bmatrix}
\begin{bmatrix}
   1 & 0 & 0 & 0 \\
   0 & 1 & 0 & 0 \\
   0 & 0 & 1 & 1/d \\
   0 & 0 & 0 & 0
\end{bmatrix} =
\begin{bmatrix}
   x & y & z & z/d
\end{bmatrix}
$$

需要注意的是，经过透视投影矩阵变换得到的坐标并没有进行投影，它只是计算出合适的 $w$ 值并把它存到齐次坐标中，而真正的投影需要等到 4D 坐标转换成 3D 坐标时才会进行。可能有的人不太理解，我明明直接进行除法运算就能够得到投影坐标，为什么要费劲弄一个齐次坐标出来？其实这主要是为了让透视投影变成一个矩阵变换，这样就能够和其它的矩阵变换一起运算。另外，实际使用的投影矩阵也并不是上面这个，因为经过投影之后 z 坐标已经没用了，在渲染管线中通常会使用 z 坐标进行深度缓冲运算。

?> 这一节所提到的所有矩阵都是对应的理想情况，如果需要向任意轴或者平面进行矩阵变换的话会要复杂很多。

### 常见矩阵变换的特性

| <center>变换名称</center> | <center>符合线性变换</center> | <center>符合仿射变换</center> | <center>是可逆矩阵</center> | <center>是正交矩阵</center> |
| :--- | :--- | :--- | :--- | :--- |
| 平移矩阵 | N | Y | Y | N |
| 绕坐标轴旋转的旋转矩阵 | Y | Y | Y | Y |
| 绕任意轴旋转的旋转矩阵 | Y | Y | Y | Y |
| 按坐标轴缩放的缩放矩阵 | Y | Y | Y | N |
| 切变矩阵 | Y | Y | Y | N |
| 镜像矩阵 | Y | Y | Y | Y |
| 正交投影矩阵 | Y | Y | N | N |
| 透视投影矩阵 | N | N | N | N |

这里要留意哪些矩阵变换是正交矩阵，因为正交矩阵的逆矩阵就等于转置矩阵，可以简化计算过程。

## Unity中的空间变换

---

### 顶点的空间变换

在 Unity 的渲染流水线中，一个顶点在映射到屏幕上的过程中通常要经过几种空间变换，如下图所示。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/vertex_conversion.png)

这里唯一要注意的是，在上述的几种空间中只有观察空间是使用**右手坐标系**的。观察空间以摄像机为原点，物体的深度越大，距离相机就越近，如下图所示。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/unity_camera_cartesian.png)

### 相机的透视投影

在 Unity 中，相机的透视投影呈现出的是一个视锥体，如下图所示。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/projection_frustum.png)

视锥体中的 `Near` 和 `Far` 分别代表的是近裁剪平面和远裁剪平面，`Field of View` 则表示的是相机视野大小。

由已知的三个数据，我们可以得到近裁剪平面和远裁剪平面的高度：

$$ nearClipPlaneHeight = 2 \cdot Near \cdot \tan \frac{FOV}{2} $$

$$ farClipPlaneHeight = 2 \cdot Far \cdot \tan \frac{FOV}{2} $$

相机的纵横比也很轻易地可以计算得到：

$$ Aspect = \frac{nearClipPlaneWidth}{nearClipPlaneHeight} = \frac{farClipPlaneWidth}{farClipPlaneHeight} $$

现在，我们可以结合之前学习过的透视投影矩阵，得到以下矩阵：

$$
\boldsymbol M =
\begin{bmatrix}
   \frac{cot{\frac{FOV}{2}}}{Aspect} & 0 & 0 & 0 \\
   0 & cot{\frac{FOV}{2}} & 0 & 0 \\
   0 & 0 & -\frac{Far+Near}{Far-Near} & -\frac{2 \cdot Near \cdot Far}{Far-Near} \\
   0 & 0 & -1 & 0
\end{bmatrix}
$$

上面展示的这个透视投影矩阵对 $x$、$y$、$z$ 进行了缩放（$z$ 坐标还进行了平移），计算时只需要将列矩阵乘在矩阵的右侧即可，例如坐标 $(x, y, z, 1)$ 的变换结果如下：

$$
\begin{bmatrix}
   x\frac{\cot\frac{FOV}{2}}{Aspect} \\
   y\cot\frac{FOV}{2} \\
   -z\frac{Far+Near}{Far-Near}-\frac{2 \cdot Near \cdot Far}{Far-Near} \\
   -z
\end{bmatrix}
$$

注意，上述的投影矩阵是针对 Unity 而言的。在 Unity 中，观察空间是右手系，矩阵变换是使用的是列矩阵右乘，并且投影变换后 $z$ 分量处在 $ [-w,w] $ 之间。DirectX 中则会有些不同，各位在学习的时候要注意区分。

顶点在经过透视投影之后，$w$ 分量不再是 1，此时我们可以用该分量来判断顶点是否处于视锥体中：

$$ -w \leqslant x \leqslant w $$
$$ -w \leqslant y \leqslant w $$
$$ -w \leqslant z \leqslant w $$

这种方式比起直接拿视锥体的六个面来判断要快得多，任何不满足上述条件的图元都需要被剔除或者裁剪。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/projection_matrix0.png)

### 相机的正交投影

正交投影的视锥体是一个长方体，因此计算要比透视投影简单。同样的，Unity 中正交投影也可以设置相关的参数：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/orthographic_frustum.png)

Unity 的正交投影矩阵如下：

$$
\boldsymbol M =
\begin{bmatrix}
   \frac{1}{Aspect \cdot Size} & 0 & 0 & 0 \\
   0 & \frac{1}{Size} & 0 & 0 \\
   0 & 0 & -\frac{2}{Far-Near} & -\frac{Far+Near}{Far-Near} \\
   0 & 0 & 0 & 1
\end{bmatrix}
$$

对于坐标 $(x, y, z, 1)$ 的变换结果如下：

$$
\begin{bmatrix}
   \frac{x}{Aspect \cdot Size} \\
   \frac{y}{Size} \\
   -\frac{2z}{Far-Near}-\frac{Far+Near}{Far-Near} \\
   1
\end{bmatrix}
$$

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/orthographic_matrix0.png)

?> 再次强调，透视投影变换并不是真正的投影，它仅仅只是计算出合适的 $w$ 分量，顶点坐标仍旧处在锥体或者立方体中。只有在屏幕映射的时候才是将顶点投影到 2D 平面上。

### 屏幕映射

在经历过透视投影、裁剪操作之后，我们就需要把三维坐标转换成屏幕上的二维坐标了。之前有提过，将齐次坐标转换成对应坐标需要使用齐次除法，说白了就是拿 $x$、$y$、$z$ 三个分量除以 $w$ 分量。通过这一步计算，我们就能够把坐标从齐次裁剪坐标空间转换到归一化设备坐标（NDC）。齐次除法得到的将是一个立方体空间，下面的两张图分别展示了透视投影和正交投影的齐次除法：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/projection_matrix1.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%9F%A9%E9%98%B5%E5%8F%98%E6%8D%A2%E4%B8%8EUnity%E7%A9%BA%E9%97%B4%E5%9D%90%E6%A0%87%E8%BD%AC%E6%8D%A2/orthographic_matrix1.png)

不过这样有一个问题，屏幕映射得到的是 2D 坐标，那么 $z$ 分量应该拿来干什么呢？一般来说，这个分量会用于深度缓冲。
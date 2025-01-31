作为一个总是去设计领域闲逛的开发，或许可以更好得把自己对设计的领悟给开发们讲清楚，这也是我输出的第一篇更加偏向于设计领域的文章，毕竟职业还是开发，只能算一个不入流的旁门左道的设计师，对设计的一些术语表述有误还请设计大佬谅解。

本文旨在给开发者一个基本的色彩的认识，把一些设计知识带入开发者的世界里，希望开发者们可以从这篇文章中受益。

## 前言

色彩理论起源是牛顿发现的三棱镜可以让光变成可以渐变的七种颜色红橙黄绿青蓝紫。由于是渐变的并且还是首位相连的那种，所以牛顿同时也发明了色轮，这个是色彩理论的最初的探索。

后来歌德完善了色彩理论，还把颜色分为了暖色和冷色。

再后来，颜色理论逐渐被发扬光大，比如蒙塞尔的球体，还有色环，明度，亮度，饱和度，色相，色调，对比度，灰度等等各种各样的发现，如果使用过ps调色的人会很熟悉这些概念，而在大多数的开发者眼里，或许颜色只是计算机里的一个16进制的数值。

很神奇这些理论的奠基者居然一个是大家眼中的物理学家数学家牛顿，另一个是大家眼中熟知的散文家，诗人《少年维特之烦恼》的作者歌德。所以说色彩不仅仅是设计师关注的，也是科学所关注的，诗人也会喜欢色彩，因为色彩传递了感情。

## 颜色的感觉

颜色来源于大自然，来源于人的肉眼所能接受到的大自然不同物体所反射的光线。所以颜色的感觉更多的也来源于大自然，比如说

绿色代表生机，由于大自然的包容，所以绿色也代表原谅。

红色会充满热情，因为红色就像火焰一样炽热，也像血液一样，所以红色有时会有种不详，后来红色也代表党。

蓝色代表冰冷也代表大海。蓝色和白色在会让人联想到冬天的雪后透彻的蓝天和白茫茫的一片。

蓝色和橙色在一起会有种落日和大海的感觉。

紫色会比较抚媚，也会让人觉得浪漫。

感受不同的颜色传达的感觉这个是一个设计师最基本的能力，也是平面设计或者视觉传达设计中最基础的知识。

## RGB

rgb通过三种不同的颜色混合来表达颜色，也是最常见的颜色表示法，混合颜色通常用三原色，即红绿蓝，用这三种颜色混合可以得到任意颜色，注意这是光的颜色的拆分，而印刷需要的是三基色，可以去搜一下三原色和三基色的区别。

### 颜色的混合

rgb分别是red green和blue，下边的这张图里可以很清晰的看到混合的过程

![1](http://)

当三种颜色都是最大的时候混合出来的颜色就是白色

绿色和红色混合是黄色（第三个块）

蓝色和红色混合是洋红色（第七个块）

绿色和蓝色混合是青色（第五个块）

橙色的话其实是较多的红色和较少的绿色混合的结果（第二个块），是处于红色和黄色的中间值。

紫色是较少的红色和较多的蓝色（第八个），等量的红色和蓝色其实应该叫做洋红色。

![2](http://)

如果熟知这些那么调色就能轻而易举了。

稍微提一下，青，洋红，黄就是三基色，颜色模式种的另外一种模式是CMYK，分别就是说Cyan，Magenta，Yellow还有Blank。开发是不会用到印刷知识的，所以不需要知道这个只是。

### 计算机中颜色的表示

计算机为了表达颜色也有一些规定，最常见的就是#开头的16进制6位字符串。

16进制是0到f，所以#fff意思是三种颜色都是最大值，所以就是白色。而#fff和#ffffff是等价的，可以认为前者是后者的缩写，比如#ffaa33就可以缩写作#fa3

而上面所示的八种颜色用十六进制就是这样

![3](http://)

知道颜色的混合就能很清楚根据给出的色值猜到大概是个什么颜色

比如#014589，拆分成rgb就是红色01，绿色45，蓝色89。由于蓝色最多，绿色次之，而蓝色加绿色是青色，那较多的而蓝色加较少的绿色所以会是一个偏蓝的青色。整体色值较低，所以偏暗一些，因此这个色值接近于藏青色。

![4](http://)

也可以根据想要的颜色，写出一个差不多的色值，比如说想要一个下边这样的桃红色

![5](http://)

这个颜色很接近洋红色，但是比洋红色更红，或者说蓝色的成分要少一些。而洋红色是#ff00ff，所以这个颜色大概就是为#ff0088。

### rgba表示

css还提供了一种表示方法是rgba或者rgb，rgba会比rgb多出一个alpha，为什么叫做alpha不叫做opacity呢。

区别在于一张图片的opacity是50%，那么意味着所有的区域都是50%的透明度。

而事实上一张图不同地方的透明度是不一样的，比如一团火焰，火焰其实在淡的区域是可以穿透的，在浓烈的地方可以认为是没有透明度的，那么这意味着不同地方的颜色的穿透能力不同，形成了一个颜色的通道，这个通道就叫做alpha通道。所以alpha可以理解为颜色的可穿透的程度。这个点到为止，反正理解成透明度也不会出错。

rgba模式和16进制本身是一摸一样的，只是16进制是00到ff，rgba是0到255，255其实是2^8。这个是因为计算机的存储颜色的方式，一个色值用8bits。

## 颜色的度

目前学习的颜色都是不同的颜色的表示方法，其实一般人所谓的颜色是指色相，什么叫做色相呢，就是说颜色的品相，通常红橙黄绿蓝靛紫就是不同的色相，就像不同的星座一样，一个范围内的某种颜色都是一种色相，和星座不同的是，色相并没有严格的规定什么范围到什么范围是什么相。因为对于设计师这种限定并没有什么意义。

既然同一种颜色其实包含各种各样的颜色，比如绿色有墨绿，青绿，淡绿等等。所以同一种颜色内，其实需要有各种各样的度。饱和度（纯度），亮度，明度，灰度，对比度。

而这些度里最重要的是色相，饱和度，明度，被称为颜色的三要素。

品颜色需要有强大的感知能力，所以在继续往下读的同时放松双眼。

### 饱和度

通俗的说饱和度降低就是让颜色显得看起来不那么刺激或者鲜艳。

**为什么饱和度高会让人产生视觉刺激强烈的感觉呢？**

是因为人眼有三种视锥细胞，用来接受三原色，最后合并为一种颜色。一个颜色对人的眼球产生刺激是因为眼睛接受到的三原色之间的差距很大，三种视锥细胞得到了差距很大的三种颜色就会刺眼，所以就产生了鲜艳感。

饱和度在色值的表现上通常是rgb三种色值的差距缩小，饱和度为0就是三原色的差距为0。而饱和度的数学概念就是三原色的最大值和最小值之间的差，上边说的那个桃红色的饱和度就是ff和00的差值是100%。

当我们把一张彩色的图片的饱和度变成0那么图片应该只有黑白灰，变得像一张黑白照片一样，但是黑白照片的算法和饱和度为0并不一样。

黑白是这个颜色的分量计算一个分量相等的灰色，一个图变成黑白之后整个图的直方图信息并不会发生改变，关于这个可以以后再聊。而饱和度为0是取rgb的最大值和最小值的中间值。让三种颜色同时趋于这个中间值。

![6](http://)

### 明度

通俗的来讲，明度就是某个颜色看起来是偏黑还是偏白。比如说上边说到的藏蓝色的整体色值都比较小，其实就是说他的明度低。如果理解了饱和度的话，在理解明度就会非常简单。

同样，我们站在生物学和数学两种角度理解这个暗还是白的问题。

生物学的角度说，暗意味着三种颜色对三种视锥细胞的刺激都很小，相反意味着三种颜色都比较多，使得看起来更白，而素色就是说明度比较高的颜色，这种颜色比较偏白，只是淡淡的展露出一种颜色，这种颜色看起来更加淡雅。

明度增加

![7](http://)

明度降低

![8](http://)

其实明度的变化也会引起饱和度的变化，因为黑色和白色的饱和度也是0。但饱和度和明度是从两个不同的维度去控制一个颜色的。这也是css中的另一种颜色表达方式HSL。

## HSL模式

工业颜色标准，通过色相 hues，饱和度 saturation 和明度 lightness来表达颜色，可以表达视觉所覆盖的所有颜色，是迄今使用最广的颜色标准，css也支持这种方式来表达颜色。

![9](http://)

这个时候很多人就开始吐槽了，是因为rgb表达的颜色不够多还是不好用，非要造出来这么多奇奇怪怪的模式，css那么多属性都学不动了，学个颜色搞得这么麻烦嘛～

其实颜色确实是门非常复杂的学问，但颜色的学问要比编程容易理解的多，因为颜色很贴近生活。而且如果你继续往下看的话，就能知道hsl可以解决问题是多么的强有力

### 使用hsl表示颜色

要用hsl，先得知道它是怎么表示颜色的呢

理论上讲一个颜色用HSL表示应该是这样的 hsl(#f00, 100, 50%)，这个表示一个饱和度100，明度在中间值不黑不白的红色。

但是这个事实上应该是这样的 hsl(0, 100, 50%)

0？0既然也能表示颜色，这个让人有点匪夷所思。

上边提到牛顿发明了一个叫做色轮的东西，这个东西就是色相。好吧，我再纠正一下，前边说到的颜色并不是真正的色相。

准确的说色相环是饱和度为100%，明度为50%的颜色组成的一个颜色环，就像下边这样。可以通过降低三种颜色值让颜色千变万化，同时也可以通过改变明度和饱和度让颜色千变万化。

而且后者更容易控制颜色的千变万化。

![10](http://)

色相环只有一个属性就是角度。0度是红色，360度也是红色，蓝色是240度，而色相环的旋转是按照上边的图逆时针转的，饱和度和明度都是通过0到100来表示百分比的。

![11](http://)

### hsl有什么用呢

要知道用处是什么，就需要知道他是怎么诞生的，hsl诞生是因为使用颜色的人并不关心颜色的值是多少，需要关心的是这个原色的变化。

比如hsl可以非常非常容易的解决千变万化的口红颜色的问题。

有女朋友的人都知道口红的颜色是一个非常恐怖的东西，但使用hsl来理解口红的颜色就非常简单，假如需要粉一些，就把色相到这转30度，需要淡一些，就把明度往高调一些。要暗一些就把颜色往低调一些。

hsl也可以很好的解决设计师配色的问题，一个视觉效果好的网页色不过三，甚至就很有可能就一种颜色。

如果两种颜色的话，通常是色相环中相差180度的两种颜色，这个叫做互补色。

如果是三种颜色很有可能是色相环的三种相差120度的颜色。

所以即便是两种颜色，或者三种颜色，只需要定一个主色，然后根据颜色来变动就好了。hsl的最大的特点就是变化。还有一点就是即便是不同的颜色，他们仍然会有相同的明度还有饱和度。

比如下边就是两个明度饱和度相同的互补色，是不是看起来比较协调统一而且还互补呢。

![12](http://)

这两个颜色用hsl表示是 hsl(30, 80, 70) 和 hsl(210, 80, 70)

但是用rgb表示的话是#f3c291和#91c2f3。相比较而言hsl可以更加清楚的看出来两者的关系，也可以配合css中的样式预处理器表示。

还有一个作用是通常表示鼠标hover active等效果的时候会由设计师给出两个不同的色值，但其实是两个明度不同的色值，所以如果使用hsl的话，可以一个颜色走天下，当然前提是这个设计师知道如何配色。

所以如果一个公司又非常好的设计规范的话，用hsl的效率要远远大于rgb

## HSV

除了hsl以外还有使用hsv模式来控制颜色的，和HSL很相似，通过色相 hues，饱和度 saturation 和亮度 brightness 来表达颜色，因为brightness有的地方会叫做value，所以hsv和hsb都是说这个模式。

ps提供了五种表示颜色的方法，分别是HSV，RGB，16进制，CMYK还有Lab。Lab是基于通道的一种模式。

![13](http://)

hsv和hsl都可以非常好的表示颜色的变化。二者的区别可以在图中很直观的看出来，前者注重明度的变化，后者注重亮度的变化。

有这种模式的原因是设计师配色主要看重明度，而修图的后期会重视亮度，所以web开发用hsl，ps中用hsv。

![14](http://)



据相关数据显示，截至2017年底，中国手机新闻客户端用户规模达到6.36亿人，移动App已经成为新闻和内容传播的最重要途径之一。而伴随着行业的竞争和发展，App中的**内容页**在提升App品质、提升使用时长及提升用户黏性等方面，扮演着更为重要的角色，同时也面临着更大的挑战。

1.	**内容页在呈现上越来越丰富。**新闻资讯作为内容页的主体，逐渐增加了更多的文字样式、内容形式、富媒体、以及广告、投票等更为丰富的元素。
2. **内容页需要更多扩展区域来提高使用时长及用户黏性。**在资讯主体之外，各个App逐渐打造了例如关注模块、推荐阅读模块、评论模块、运营模块等越来越多的扩展阅读区域。
3. **短视频、直播的争夺越来越激烈。**越来越多的新闻App都将视频作为独立的模块和独立的内容页进行展示。
4.	**同质化产品竞争激烈。**要求更快的迭代速度、更优质的用户体验、更小的实现成本。

所以，新闻类App**内容页**架构的设计和技术的优化，也要配合产品形态的发展，在越来越复杂的需求挑战下，拥有快速响应的能力和稳定优质的体验。

本文结合分析目前主流（DAU）新闻类App如`今日头条、腾讯新闻、天天快报、一点资讯等`内容页技术方案的选择，一起探索新闻类App内容页的技术实现和优化。

<br>

> ***
>_插播广告 —— 几十行代码完成新闻类App多种形式内容页_ 
>
>_[HybridPageKit](https://github.com/dequan1331/HybridPageKit) ：一个针对新闻类App高性能、易扩展、组件化的通用内容页实现框架。_
>
>_基于[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)、[WKWebViewExtension](https://github.com/dequan1331/WKWebViewExtension)、以及本文中关于内容页架构和性能的探索。_
>
>***

<br>

## 概念定义

结合目前主流的内容页实现方式，我们把内容页分为上下两个部分，为了方便后续的阅读先简单定义下关键的名词。

1.	上部分通常用WebView实现。常规包括标题 + 作者（关注）+ 资讯内容，我们称为**`WebView内容区`**。
2. 	下半部分主要是平行于WebView的各种扩展内容，常规包括点赞打赏、广告推广、相关推荐，热门评论等等，我们称为**`Native扩展区`**。
3. 	WebView中每个复杂UI呈现、扩展区中每个独立模块，我们都称为一个**`模块`**或**`组件`**。


完整来看，整个内容页右侧（右滑）普遍为评论页。无论是之前流行的ScrollView右滑还是近期流行的Push新页面，这两种方式实现起来都比较简单且较为独立，故本文暂时忽略右侧（右滑）评论的部分。

## 目录

<center><img width="80%" height="80%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/index.png"></center>

## 技术方案选择

### 1.WebView类型选择
	
不同于微博，新闻类App的内容以段落性的文字为主，配合段落间的图片、富媒体等。同时为了满足跨平台的一致呈现、PC网页的文章转载、不同平台文章的抓取，以及注重阅读而非交互等原因，使用**WebView**加载渲染本地的HTML字符串数据已经成为了新闻类App通用的方案。

**UIWebView ~~VS~~ WKWebView**

- 稳定性:
	
	UIWebView较多的WebCore、JavaScriptCore Crash，以及系统性的内存泄露导致OOM，对整个App的稳定性都是极大的隐患。反观WKWebView，基于独立进程，不会占用App的内存计算，同时也不会导致主App Crash。所以在系统级的稳定性上，WKWebView有着极大的优势。

- 	加载速度:
	WKWebView通过JIT大幅优化了JS的执行速度，但是对于新闻类App内容页的使用场景来说，简单的进入、退出页面，且单纯的加载渲染HTML字符串，WKWebView比UIWebView慢了很多（`Benchmark`）。
	
-  	兼容性:
	NSURLProtocol的无法使用、长按MenuItems Bug（before iOS11）、iOS8不能删除Cache、设置Cookies及UA、POST参数、异步执行JS...这一系列的问题，成为了稳定项目替换WKWebView最大的挑战。
	
-  	扩展性:
	WKWebView具有更加丰富的接口、更多HTML和CSS的支持、以及更加友好的JS交互。同时Api的持续更新和社区的活跃，从长远使用的角度看有着极大的优势。

**修复、扩展WKWebView**

通过以上的分析，WkWebView从系统级的稳定性、性能以及后续扩展性都有很大的优势。通过[WKWebViewExtension](https://github.com/dequan1331/WKWebViewExtension)扩展修复原生WKWebView，结合[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中WKWebView的回收复用逻辑，极大程度上解决了原生WKWebView的问题，起到了很好的效果。

- 修复扩展的问题:
	通过逐阶段分析耗时，在内容页的使用场景下，WKWebView从alloc到准备开始渲染这段时间，有着极大的优化空间。在浏览内容页这种场景下，[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中通过WKWebView的复用回收以及资源缓存，极大降低了WKWebView加载渲染HTML的时间，使之低于原生UIWebView。

	通过私有方法的扩展和代码优化，在[WKWebViewExtension](https://github.com/dequan1331/WKWebViewExtension)中支持了URLProtocol、修复了MenuItems的bug、支持iOS8清理缓存、扩展安全的JS执行方法、以及扩展NavigationDelegate以兼容JSBridge逻辑等。

- 	无需解决的问题:

	对于新闻类App内容页的使用场景，一些WKWebView的问题并没有必要形成通用的解决方案以兼容代码。比如POST请求不能带参数、Javascript异步执行等问题，都可以通过代码的重构来进行解决。尤其不推荐卡主Runloop从而同步JS的方式。

-  	遗留问题:

	目前，在使用WKWebView的过程中，唯一未解决的问题就是可靠、全面的白屏检测方案，从而支持WKWebView在任何情况下的Crash进行重载。诸如系统Crash回调、WebView Title监听、ContentSize监听、甚至屏幕随机取色值等方法都不能满足全部的白屏场景。


### 2. 	WebView内容区与Native扩展区的衔接

对于目前的主流App来说，单纯的WebView已经无法满足复杂的呈现和逻辑。如何在页面中合理的处理WebView与扩展区中的多种View协同滚动，灵活扩展，并且支持下拉刷新、上拉加载等操作，不同的新闻类App也有不同的技术方案。

**结合TableView**

<center><img width="50%" height="50%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/tableView.png"></center>
	
-	实现原理:

	由于扩展区中列表类型的模块较多（例如相关文章、评论等），最简单的实现即Native扩展区的模块拆分到Cell的粒度，整体使用TableView实现。对于扩展区和WebView的衔接，如上图一般有两种实现方案：TableView根据WebView的Inset（或Div占位）插入到WebView中 & WebView作为TableView的Header。

-	优点:

	这种方法相对简单，容易实现内容页各个模块的布局，同时基于TableView的刷新逻辑，也能动态的处理各个模块的更新、插入删除，并且支持家在更多等。和WebView的结合滚动也较为流畅。

-	不足:

	这种方式将Native扩展区的模块粒度都区分到Cell的层级，列表类型模块只能通过Cell或者以Section的模式进行管理，同时也无法跨页面的整体复用UI及业务逻辑。UI的布局依赖TableView模式，灵活性较差。随着组件类型的增多，非同质性的View也没有充分利用TableView的复用。
	
	同时无论使用哪种方式和WebView衔接，都影响了WebView、TableView的独立渲染展示，增加了维护的困难。并且Header与Inset对于头部区域的扩展，如下拉刷新等，实现都较为困难。

**ScrollView嵌套**

<center><img width="70%" height="70%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/Scroll.png"></center>

-	实现原理:

	这种实现用一个ScrollView作为Container，将WebView及扩展区的组件分别作为SubView。全部SubView禁止滚动，内容页的全部滚动都发生在Container上。对于SubView中的滚动视图，如果ContentSize小于屏幕高度，则作为普通View，否则设置为屏幕高度，通过offset和Frame的计算，动态的调整视图相对Container的Frame以及自身的ContentOffset，实现滚动效果。

-	优点:
	
	这种方式完全独立每个模块的实现，使UI和业务逻辑一一对应。对WebView的渲染没有干扰，模块的加载和布局灵活管理、复用，模块业务逻辑独立内聚。添加删除模块、实现上拉下拉等操作简单。极大的提高了灵活性和复用的可能。

-	不足:

	由于这种方式需要对SubView中的滚动视图进行计算、模块动态更新时整体布局也需手动刷新等，极大的提高的实现的复杂度。
	
	基于[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)，在[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中，封装了以上ScrollView嵌套逻辑。这样就隐藏了复杂的实现逻辑和边界条件，充分的保留了灵活性的特点。同时对于内容页的使用场景，精简了嵌套滚动的使用，扩展上拉加载更多及下拉刷新逻辑，使整个方案实现简单、灵活扩展。


### 3. 	WebView内复杂UI、复杂交互模块的展示

随着核心的WebView内容区逐渐支持复杂的呈现方式，单纯的H5基础渲染已经满足不了现有的需求，比如视频的交互、音乐的续播、以及各种地图、投票等组件。同时Web中复杂的UI和逻辑也极大降低了WebView的渲染速度，增加了开发和维护的成本。

**复杂UI及逻辑实现困难**

-	为了满足更好的交互体验，资讯内容中富媒体内容逐渐增多，如视频的续播、小窗播放、音乐悬浮播放、内容中插入地图、投票等。同时随着产品功能的迭代，例如图片类型的简单模块，也增加了点击全屏、长按保存、二维码识别、双击扩大等交互。这些复杂的UI和逻辑导致CSS和JS增多，Native和Web的通信增加，以及大量运用LocalStorage等浏览器存储，增加了客户端开发和维护的成本。

**简单图片的展示耗时**

-	对于内容WebView中的图片，最简单的作法，就是后台直接下发Img标签，依靠WebView自身的下载与渲染。但是这种方式灵活度较低、客户端无法合理的控制下载时机、无法做自定义的缓存以及裁剪等。
-	对于简单Img标签的升级，即后台数据单独下发图片数据，客户端根据需求自定义选择下载时机及缓存策略。Html模板中先用占位图占位，Native下载成功后替换标签的Src进行展示。这种方式虽然解决了灵活性的问题，但是也带来了整个流程的复杂性，以及多次IPC间的通信延迟。
-	为了兼顾灵活性，以及缩短图片的Loading时间，我们在单独处理图片的同时，替换内容WebView中全部图片为Native，减少不必要的流程及通信，极大提高了加载的速度。

**Native化全部非文字类组件**

为了减少实现复杂UI、复杂交互模块的开发、维护成本、减少模块在Web和Native间的逻辑流程，提高Web中模块的加载展示速度，在[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中将Web中全部非文字类模块全部Native化。
	
<center><img width="70%" height="70%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/div.png"></center>

-	页面模板使用空div占位:

	结合后台的模板与数据，全部模板中全部非文字类的组件，映射成统一Class的Div，通多唯一的id与数据绑定。组件默认实现占位图逻辑，对于同步数据同时设置组件的Size，异步数据则先设置为0。替换后WebView对模板进行渲染。

-	渲染完成通过JS获取位置:

	WebView渲染成功回调，通过JS获取全部统一class对应WebView的Frame，以及对应的唯一Id。
-	在相应位置粘贴NativeView:

	在进行以上两个步骤的同时，进行下载图片数据、NativeView创建、初始化、异步数据拉取等工作。在JS回调全部位置时，根据位置及ID，粘贴Native组件。

-	调整字体大小，组件异步数据拉取：对于异步的变化，在复用逻辑之后，下文将结合一并说明。

### 4. 内容页全部组件的滚动复用

在Native化全部非文字类组件之后，面对文章中图片、富媒体数量的增多，以及Native扩展区元素的增加，没有复用回收的内容页从滚动性能及内存两个两个方面都面临着挑战。同时，为了更好的提升用户体验，需要对各个组件滚动时的位置进行计算，从而区分不同的区域进行诸如预处理、延迟释放等逻辑。

**主流滚动复用框架**

-	继承特殊ScrollView:

	目前流行的框架如alibab的[LazyScrollView](https://github.com/alibaba/LazyScrollView)，对于实现复用回收机制，都需要继承相应的ScrollView，这种方式对于WKWebView来说，是无法实现的。

-	继承特殊Model:

	由于滚动复用需要保存View对应的数据信息，大部分开源框架需要继承特殊数据Model，生成对应必要的参数或方法，对于支持多种类型组件的通用框架来说，继承的实现方式不易于扩展和维护。
	
-	View滚动状态简单:

	滚动时位置的计算，最简单的方式就是根据屏幕的高度计算是否进入屏幕，对于预加载的需求，绝大部分开源框架也是只是在屏幕区域的上下增加了Buffer，仍然不能区分具体的状态，如进入buffer、进入屏幕等，无法满足复杂的业务逻辑。

**WebView中组件的滚动复用**

<center><img width="60%" height="60%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/scrollData.png"></center>

-	无需继承:

	在[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)中，为了兼容WebView、ScrollView等一切滚动视图中子View的复用回收，我们通过scrollView delegate的扩展分发，扩展handler单独处理子View的复用回收，这样就在无需继承的前提下，支持所有滚动视图中子View的复用回收。

-	数据驱动:
	
	由于View需要不断的复用回收，所以数据、状态、位置、对应的View类型都存储在对应的Model中，不但实现了数据驱动易于动态扩展，同时优化了复用的逻辑，也缓存住了Frame等关键信息优化了渲染布局逻辑。
	
-  	面向协议:

	由于滚动复用的模块对应的View及数据Model种类众多，在不动态扩展NSObject、UIView的情况下，无法做到通用的逻辑公用。所以为了更好的支持扩展、更灵活的实现方式，[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)中面向通过扩展数据Protocol，使得任何Model轻松实现复用回收对应逻辑。
	
-  	更加丰富的状态:

	在[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)中，为了满足更复杂的需求，如视频预加载及自动播放、Gif预加载及自动播放等，我们扩展了组件在滚动过程中的状态，增加自定义workRange，使组件在滚动过程中的状态变为3种，即None、prepare区域及Visible区域，更加全面准确的记录状态切换，更加灵活的支持业务场景。同时通过3种状态扩展为二级缓存，对View在不同级别的缓存设置不同的策略。
	
综上，通过[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)只需将模块对应Model扩展增加协议，滚动视图扩展Delegate，就可实现任何滚动视图中子View的回收复用功能。

**内容页中全部组件的滚动复用**

在解决了内容WebView中非文字类组件的Native化、滚动复用之后，我们将实现思想运用到包含Native扩展区的，内容页整体架构中。如果从内容页的维度去看，内容WebView也可以算作一个组件，它和扩展区的各种组件一起作为Container的子View，也可以运用上面提到的[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)进行实现和管理。
	
<center><img width="30%" height="30%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/Rns2.png"></center>

所以整个内容页就是从两个维度、运用[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)中的实现方法两次实现滚动复用回收、数据驱动、组件自管理以及组件状态切换逻辑。
	
### 5.	组件异步拉取与动态调整

面对复杂的需求、以及按需加载、异步拉取等优化体验的策略，在[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中也针对相应的场景做了高效的处理。

**WebView字体大小调整**

当WebView中字体大小调整时，需要同时调整全部Native组件的位置。我们监听WebView的ContenSize变化，当变化发生时，重新执行获取组件位置的JS语句获得全部组件的新位置。基于滚动复用的逻辑，只需要对在屏幕中的组件View的位置进行调整，其余只需要重新对组件对应Model的Frame进行赋值，极大提升了效率。在此基础上，要动态的检测ContenSize是否小于屏幕高度，高度小于一屏幕时，要同时调整Native扩展区组件的位置。

**WebView中组件异步拉取数据渲染**

对于异步拉取数据的组件，由于初始化时占位Div的高度为0，当数据获取成功，并渲染好组件后，需要首先执行JS动态修改对应占位Div的大小，之后按照以上的逻辑，重新赋值Native组件位置。

**Native扩展区组件异步拉取数据渲染**

Native扩展区中的组件不同于WebView中的组件，不依赖WebView自身渲染。所以当动态调整大小时，之需调整全部Native扩展区组件数据Model中保存的Frame信息，同时调整在屏幕中的组件位置即可。

## 内容页组件化架构

在实现了以上技术关键点的基础上，如何合理的设计内容页通用的架构，快速响应内容页的各种需求调整，使整体架构易扩展、易维护，同时有较高的性能及较小的内存占用，成为了整个内容页架构实现的重点。在[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中，我们围绕灵活复用、高内聚低耦合、易于实现扩展三个重点的方向，设计实现了基于组件化的内容页整体架构。

### 1.	组件化解耦及组件通信

为了满足内容页业务的相对独立，支持快速响应迭代及组件整体复用，内容页整体的结构应满足通用性、易于扩展、以及高内聚低耦合的特点。所以在[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)的支持下，采用组件化的方式实现全部内容页业务模块。

**组件化解耦**

为了达到组件的高内聚、与内容页的低耦合，在[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中拆分业务逻辑为独立的组件化的处理单元，每个处理单元通过MVC模式实现。其中Model作为组件的数据，只需要在实现解析逻辑同时，实现对应delegate即可。Controller只需要实现组件间通信的delegate，选择性的实现例如controller生命周期、webview关键回调、以及滚动复用相关的方法即可。通过组件的自管理及复用，组件可以集成统一的上报逻辑、业务逻辑到自己的Controller中，并且在不同类型的页面灵活复用。

**组件通信**

为了更好的实现组件化的结构，组件的Controller需要在内容页初始化时进行注册。内容页在每个关键的生命周期或业务节点，采用中心化通信，广播执行相应的方法，组件的Controller按需实现处理即可。对于新增、删除功能，只需扩展delegate中的方法，内容页中触发方法、组件中实现方法即可。

<center><img width="60%" height="60%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/componentComm.png"></center>


### 2.	组件及WebView的复用管理

**WebView & 组件View全局复用**

为了提高WKWebView渲染速度，通过建立全局WKWebView复用回收池来复用WKWebView。除了基本的线程安全、复用状态管理等，在进入回收池前要load特殊Url以维护整个backFowardList。组件的View也是通过全局的复用回收池进行管理，使得相同的组件View可以灵活的出现在内容页、列表页等App内各个页面，极大的减少了开发成本，提高运行效率。

**自动回收 & 内存管理**

WebView及组件View实现自动回收逻辑，每次在申请新View时检测活动队列中View的SuperView是否为nil，是则自动回收防止内存泄露，同时增加View最大数量阈值、内存告警自动释放逻辑等。

### 3.	内容页整体架构

**易于扩展业务节点 & 组件类型**

对于增加关键的业务节点用于组件业务处理，我们只需扩展delegate中的方法，在相关组件中实现。内容页Controller中在相应位置，通过统一函数触发广播代理方法即可。对于增加组件来说，只需创建组件完全独立的MVC代码，实现数据解析Model并实现滚动复用delegate，在组件Controller中实现delegate中需要的方法等待调用，以及初始化时在内容页注册即可。删除组件完全无需操作内容页，删除独立的MVC结构并停止注册即可。

**易于扩展内容页类型**

为了实现内容页扩展区的灵活复用，在[HybridPageKit](https://github.com/dequan1331/HybridPageKit)中也扩展了非WebView类型的内容页。就像文中之前提到的，如果将WebView看做一个整体作为一个组件，基于[ReusableNestingScrollview](https://github.com/dequan1331/ReusableNestingScrollview)的位置动态管理，完全可以替换成普通的View（类似Banner视频内容页），或者可扩展收起的View（问题回答页面）甚至tableView等。所以整个App内各种类型的内容页只需要简单的配置，便可进行实现和组件复用。

<center><img width="80%" height="80%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/pageType.png"></center>

**内容页架构**

结合 `ReusableNestingScrollview`、 `WKWebViewExtension` 以及组件化的设计思路，`HybridPageKit` 整体的架构如下：

<center><img width="70%" height="70%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/hybrid.png"></center>

通过继承特殊的内容页Controller并进行简单的配置，即可生成不同类型的内容页整体架构。框架内集成基本的Mustache解析和渲染。结合后台数据，只需实现对应页面中组件MVC逻辑即可。其中Model只需实现对应Protocol，Controller在内容页中注册，实现对应Protocol即可。

## 首屏加载速度优化

新闻类App内容页，在Native的页面框架下，基于WebView进行加载和渲染。所以，从优化的角度就延伸出两个维度，即从Web的维度优化，以及从Native的维度优化。

### 1. Web维度的优化

-	WKWebView的复用 : 

	通过WKWebView的复用，极大的缩短了WebView从创建到渲染结束的时间。
	
- 	利用HTTP缓存 : 

	对于内容WebView中必要的CSS以及JS，以及必要的基础Icon，可以通过设置HTTP缓存，依靠浏览器自身缓存提高效率。同时通过资源md5校验以保证刷新资源。
	
-  减少资源请求并发 : 

	通过Native化全部非文字类的内容，Web页面只加载最近本的Html内容，减少了业务逻辑的资源请求和并发。
	
- 	减少Dom & Javascript复杂度 : 

	通过Native化全部非文字类的内容，极大的减少了Dom的复杂度、CSS的复杂度以及过多的JS业务逻辑。
	
-  其它Web优化通用方法 : 

	精简Javascript，使用iconFont，CSS & Javascript文件压缩等

### 2. Native维度的优化

-	数据模板分离，资源并行加载 :

	基于后台数据以及Native化组件，内容页Html中模板与数据分离，使得全部资源如图片视频等都可以通过Native在合适的时机异步并行加载。不依赖与Web的渲染。

-  预加载数据,延迟加载组件:

	对于内容页关键内容（Webview）的拉取，大部分App都放到了列表页中进行。进入内容页时直接从Cache中取出内容模板，直接交给WebView渲染。基于 `ReusableNestingScrollview` 扩展丰富的状态及二级缓存，在页面滚动的过程中各个组件也可以精确的实现按需加载、预加载等逻辑。

-	Native化非文字UI，及组件化实现负载均衡 :

	WebView中非文字类UI Native化，极大的缩短了展示所需的流程，减少了进程间通信，减少了I/O及图片编解码逻辑，提高了类似图片类的UI展示速度。
	
	组件的解耦与自管理，以及广播delegate的实现，为组件的按需加载、按优先级加载提供了基础。对于内容页的各个组件来说，在内容页展示之前大部分是不需要初始化、数据拉取以及渲染的。组件化之后的组件可以根据业务优先级，在不同的关键生命周期回调中实现业务逻辑，以减轻内容页创建、模板拼接以及WebView渲染的压力。简单的举例，由于内容WebView几乎都大于一屏，扩展区中的全部组件都可以在WebView渲染结束后进行View创建、网络拉取和渲染等，这样即不影响用户的使用，同时极大的释放了渲染结束前的网络、IPC及CPU压力，提高首屏展示速度。
	
	<center><img width="80%" height="80%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/need.png"></center>
<br>
- 	组件的滚动复用 & 全局复用 & Model缓存Frame:

	基于 `ReusableNestingScrollview`扩展数据Model，缓存对应View的Frame信息，结合View的滚动复用，极大的减少了UI布局的逻辑和计算。页面内组件的滚动复用及页面间的组件复用，也同时减少了组件View的初始化耗时。

-	其它通用方法:

	基于App的技术实现和业务逻辑的优化，如异步执行业务逻辑、 图片编解码优化及资源缓存，DNS缓存等。

### 3. 整体优化方法

综上，从一个内容页在列表上的点击，到WebView渲染结束，最后到用户的滚动操作，按照时间的顺序，全部的优化策略如下图：

<center><img width="90%" height="90%" src="https://raw.githubusercontent.com/dequan1331/dequan1331.github.io/master/assets/img/opt.png"></center>

## 拾遗及Tips

对于新闻类App内容页的完整的解决方案，还有一些基本的技术点，比如模板引擎及模板拼接的模块、JSApi注入及管理的模块等等，由于篇幅所限，暂且不做深入的展开。

-	新闻类App的内容页，除去基本的渲染HTML数据外，同时也需要支持服务于活动、运营的临时H5页面。这些页面为了和Native进行交互，在自定义JSApi注入、JSBridge的选择、后台下发domain黑白名单、以及相关的安全性考虑也是整个实现中重要的一环。同时由于WKWebView支持复用回收，加载本地Html类型的WebView应该与加载H5的WebView在不同的回收复用池分开管理。

-	对于内容页图片的管理，绝大多数App都将之纳入了App统一的图片管理体系中。无论使用哪个开源图片库，在缓存策略上，尽量将内容页图片的缓存策略与其他的有所区分，或者使用`LRU + FIFO`的缓存策略，避免进入内容页大量图片占用缓存空间，导致列表图片释放。同时从使用的角度来说，重复进入同一篇文章的场景也不会频繁的出现。

-	由于各个App的数据接口和技术选型不同，在 `HybridPageKit` 中只简单的实现了基于Mustache的模板拼接，主要是由于它的logic-less、多终端集成的方便以及开源社区的活跃。对于这部分逻辑，需要根据后台数据的格式及业务需求自定义的扩展。


内容页整体的实现和优化，依赖整个App的技术实现和结构，在实现和优化的过程中，还有许多权衡和妥协，以及许多通用的、细节的优化，这里就不一一赘述。

<br>

## 写在最后

文章全部的探索及分析的实现，除对应业务逻辑外，应用封装成三个框架：`HybridPageKit`、`ReusableNestingScrollview`以及`WKWebViewExtension`。最终可以通过几十行代码，完成新闻类App多种形式的、高性能、易扩展、组件化的内容页实现。

有任何疑问，欢迎提交 issue， 或者直接修改提交 PR!


* Benchmark https://github.com/dequan1331/WebViewBenchMark
* HybridPageKit https://github.com/dequan1331/HybridPageKit


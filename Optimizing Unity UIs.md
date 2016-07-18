[TOC]

# Optimizing Unity UIs

## 01 A guide to optimizing Unity UIs

​	当我们需要开始优化UGUI的性能问题时，需要注意的核心问题是Drawcalls的平衡，尽量的减少DC的数量。利用一些技巧来是UI的Drawcalls的数量较少到最少。

4类可能造成UI性能问题：

* 过多的GPU fragment shader使用(例如，过高的fill-rate)；
* 花费过多的时间在rebuild一个Canvas batch；
* 一次rebuild太多的Canvas batches(over-dirtying);
* 花费太多的时间在生成verties(通常是text等)；

注意：

1. 原则上，性能由发送给GPU的Drawcalls的数量造成；
2. 实际上，GPU的过载更可能受到fill-rate的边界限制；

## 02 Fundamentals of Unity UI

### 2.1 Term

**Canvas:**负责将geometry合并到batches，并且生成合适的渲染命令以及发送batches到Unity的Graphics系统。这些工作都将有native C++代码完成，称做rebatch。当Canvas的geometry请求rebatching时，Canvas被认为是*dirty*。

**Geometry:**通过*Canvas Renderer*组件向Canvas提供绘制的数据。

**Sub-canvas:**一个嵌套在另一个Canvas的Canvas。sub-canvas将隔离它的子节点与父canvas之间的联系。一个被标记为*dirty*的sub-canvas不会强制它们的父canvas重建geometry数据。与之相反，父canvas的重建可能会要求sub-canvas重建。

**Graphic:**Unity UI使用c#写的所有提供绘制geometry一个基类。大多数UGUI的Graphics都通过`MaskableGraphic`子类实现。该子类实现`IMaskable`接口达到能够被mask。

**Layout:**控制`RectTransform`的大小和位置，并且通常用于创建复杂布局。Layout仅仅依靠`RectTransform`,通过修改其属性来实现布局；不依靠Graphics类，能够用于UGUI的Graphics组件。

**CanvasUpdateRegistry:**Graphics和Layout都依赖它，它不会暴露于Editor模式下。它能够追踪那些标记为必须更新的Graphics和Layout组件，并且当需要时触发更新（与其关联的Canvas调用`willRenderCanvas`事件）。Layout和Graphics通过*rebuild*更新。

### 2.2 Rendering Details

所有的geometry都将通过Canvas绘制在Transparent渲染队列中，即通过UGUI产生的geometry将永远是从后往前绘制并带有alpha blending。多边形的每一个像素被栅格化时都将被采样，即使它们整个都被其他不透明多边形等覆盖。在移动设备上，高等级的overdraw会迅速的超出GPU的fill-rate能力。

### 2.3 The Batch Building Process(Canvases)

通过Canvas合并UI元素的mesh并且生成合适的渲染命令发送给Unity图形渲染管线的过程称之为batch building。这个过程的结果是缓存和重用直到Canvas被标记为dirty，这时将rebuild这个Canvas。

用于Canvas的mesh来自于附着在这个canvas上的canvas renderer组件，但是并不包括任何sub-canvas。

构建的batch的过程中会依据mesh的depth，overlap以及shared meterials等等信息来对mesh进行排序。排序的过程是多线程的，所以在不同的CPU上得表现是非常不同的。

### 2.4 The Rebuild Process(Graphics)

Rebuild的过程发生在layout以及mesh重新计算的过程中，它将在`CanvasUpdateRegistry`中执行。在`CanvasUpdateRegistty`有`PerformUpdate`方法，这个方法将在Canvas组件调用`WillRenderCnavases`事件时调用，这个事件每帧执行一次。

`PerformUpdate`执行的三个步骤：

* 通过`ICanvasElement.Rebuild`方法重建被标记为dirty的layout组件；
* 通过`ClippingRegistry.Cull`方法实现任意注册的Clipping组件(例如Masks)裁剪任意被裁剪组件；
* 被标记dirty的Graphics组件请求重建它们的graphics元素；

对于Layout和Graphics的重建过程将被分割到多个部分。Layout的重建运行在三个部分包括:PreLayout,Layout,PostLayout,而Graphics的重建则运行在两个部分包括:PreRender,LatePreRender。

#### 2.4.1 Layout rebuilds

为了重计算包含在一个或多个Layout组件中的components的合适位置和大小，必须在它们合适的层次顺序中应用Layout。越靠近GameObject的root层次的Layout，会优先计算。为了做到这个，UGUI通过在hierarchy中的depth来排序被标记为dirty的Layout组件。depth越大，优先渲染与重建。

#### 2.4.2 Graphic rebuilds

UGUI通过调用`ICanvasElement.Rebuild`方法来控制Graphic组件的rebuild过程。

两种不同的rebuild过程：

* 如果vertex数据被标记为dirty(当组件的RectTransform改变大小)，那么相应的mesh也将rebuild；
* 如果material数据被标记为dirty(当组件的material或者texture改变)，那么它们附着的Canvas Renderer的material将被更新；

Graphic Rebuild过程不会发生对Graphic组件的任何排序操作，也不通过任何顺序的列表。

## 03 Unity UI Profiling Tools

### 3.1 Unity Profiler

使用Profiler分析UI性能问题时，主要观察`Canvas.BuildBatch`和`Canvas.SendWillRenderCanvases`的变化；

`Canvas.BuildBatch`是执行Canvas Batch build过程的情况，执行在native-code层的计算消耗。

`Canvas.SendWillRenderCanvases`包含C#脚本对于Canvas组件的willRenderCanvases事件的调用。Unity UI的CanvasUpdateRegistry类接收这个事件并且使用它运行重建的过程。在这个过程中任何包含被标记为dirty的UI组件的Canvas将会被update。

为了更容易看到不同的UI性能，我们可以从Profiler的分类中disable来自“Rendering”和“Scripts”的追踪。

### 3.2 Unity Frame Debuguer

使用FrameDebugger工具主要是为了观察UI的drawcall的数量以及分布是否合理，并减少不合理的drawcall消耗。

Unity UI的drawcall的位置依赖于Canvas的RenderMode的选择：

* Screen Space-Overlay将出现在Canvas.RenderOverlays组；
* Screen Space-Camera将出现在Camera.Render组下的Render.TransparentGeometry；
* World Space则将出现在每个World Space Camera下的Render.TransparentGeometry下；

注意：所有的UI的drawcall都会被“Shader:UI/Default”标识。

不合理的drawcall的主要原因还是因为无意识的UI叠加造成的，我们通过修改调整UI，然后观察FrameDebugger中UI的drawcall的变化，来确认UI布局是否合理。

所有Unity的UI组件生成一些列的quads来标识它们的geometry。而大多数UISprite或者Text glyphs仅仅只占用少部分的quads用来表示它们，剩余的都是空的空间。结果：UI设计师无意识的叠加多个来自于不同material的quads，导致不能够combine batched。

所有的UI操作都在transparent queue中实现，所以如果同材质的quads中间插入了不同材质的quads，那么将破坏同材质quads的combine操作，因为不同材质的quads必须draw在同材质的quads的中间。

### 3.3 Instruments & VTune

Xcode的Instruments和Intel的VTune能够深度的分析Untiy UI的rebuild和Canvas的batch计算。

* Canvas::SendWillRenderCanvases是一个C#的Canvas.SendWillRenderCanvases的C++Wrapper类方法，用于执行Rebuild操作的过程；
* Canvas::UpdateBatches与Canvas.BuildBatch完全等价，执行实际的Canvas的Build Batch操作。

当app时采用il2cpp构建时，我们可以采用以上工具来观察UI的性能消耗：

* IndexedSet_Sort和CanvasUpdateRegistry_SortLayoutList用于Layout重计算前排序所有标记为dirty的Layout组件。
* ClipperRegistry.Cull将调用所有实现了IClipRegion接口的实现。内置的实现包括RectMask2D。在ClipperRegistry.Cull调用期间，RectMask2D组件将循环所有clippable元素。
* Graphic_Rebuild包含实际计算代表Image,Text或者其他Graphic-derived组件的mesh的消耗。
* Graphic_UpdateGeometry:Text_OnPopulateMesh,当使用BestFit时，消耗将比较明显。Mesh的修改，例如Shadow_ModifyMesh和Outline_ModifyMesh等，计算这些效果的计算消耗能够通过该方法查看。

### 3.4 Xcode Frame Debugger & Intel GPA

Low-level frame debugger工具本质上是从单个batched的UI消耗以及UI overdraw的消耗来分析性能的瓶颈。

#### 3.4.1 Using the Xcode Frame Debugger









## 04 Fill-rate,Canvases and Input

为了较少GPU’s fragment管线的压力可以采取两个措施：

* 减少复杂fragment shader的使用，采用简单的shader替代；
* 减少必须sampled的像素的数量；

因为UI shader通常都是标准化的，所以通常大多数问题在于过多的fill-rate的使用。这是由于存在大量的UI元素的叠加或者多个UI元素占据大量屏幕空间所引起的，这两者将引起高级别的overdraw。因此，我们需要减缓fill-rate过度使用以及减少overdraw.

### 4.1 Eliminating invisible UI

对于已经存在的UI元素该方法是最小的改动。最通常的做法就是使用一张不透明的背景图遮罩后面的所有UI，所有在背景图下的UI都将不可见。

### 4.2 Disabling invisible camera output

如果使用的是full-screen的UI，此时world-space camera仍然渲染3D场景，这是没必要的。当一个full-screen的UI遮罩后面的场景时，我们需要disable场景的camera以较少GPU的渲染压力。

注意：如果canvas被设置为"Screen Space-Overlay"那么它不考虑场景中激活的camera的数量。

### 4.3 Majority-obscured cameras

实际上，大部分UI不可能完全的遮挡住3D场景，但是至少会遮挡住大部分场景。在这种情况下，最好的优化是考虑将可见场景部分绘制到render texture中。如果将场景中可见部分“cached”到render texture中，那么可以将world-space camera禁用，并且“cached”的render texture可以绘制到UI之下来faked实际的场景。

### 4.4 Composition-based UIs

考虑一个简单的带有背景，一个文字按钮的UI，GPU必须sampled背景图，然后是按钮的texture，最后是text的atlas texture，总共三次sampled。当UI的复杂度增加时，GPU的sampled的数量将会飞速的增加。所以一般我们会考虑将一个UI中的texture按合适的layer来合并texture到一张atlas中，以达到较少GPU的sampled。但是需要注意不能复用的texture合并将增加内存的使用。

### 4.5 UI Shaders and low-spec devices

UGUI内置的shader支持masking，clipping以及大量其他复杂的操作。由于增加的多种功能，所以比起Unity2d shader在低端设备上(例如iPhone4)执行性能较低。

如果我们不需要masking，clipping等等功能，我们可以编写一个高性能的自定义shader。

```c#
Shader "UI/Fast-Default"
{
	Properties{
  		[PerRendererData]_MainTex("Sprite Texture",2D) = "white"{}
      	_Color("Tint",Color) = (1,1,1,1)
	}  
  	SubShader{
  		Tags{
  			"Queue" = "Transparent"
            "IgnoreProjector" = "True"
            "RenderType" = "Transparent"
            "PreviewType" = "Plane"
            "CanUseSpriteAtlas" = "True"
		}
      	Cull Off
        Lighting Off
        ZWrite Off
        ZTest [unity_GUIZTestMode]
        Blend SrcAlpha OneMinusSrcAlpha
        
        Pass{
  			CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            #include "UnityUI.cginc"
            
            struct appdata_t{
  				float4 vertex:POSITION;
              	float4 color:COLOR;
              	float2 texcoord:TEXCOORD0;
			};
          	struct v2f{
  				float4 vertex:SV_POSITION;
              	float4 color:COLOR;
              	half2 texcoord:TEXCOORD0;
              	float4 worldPosition:TEXCOORD1;
			};
          	fixed4 _Color;
          	fixed4 _TextureSampleAdd;
          	v2f vert(appdata_t IN)
            {
  				v2f OUT;
              	OUT.worldPosition = IN.vertex;
              	OUT.vertex = mul(UNITY_MATRIX_MVP,OUT.worldPosition);
              	OUT.texcoord = IM.texcoord;
              	#ifdef UNITY_HALF_TEXEL_OFFSET
                OUT.vertex.xy += (_ScreenParams.zw - 1.0) * float2(-1,1);
                #endif
                OUT.color = IN.color * _Color;
              	return OUT;
			}
          
          	sampler2D _MainTex;
          	fixed4 frag(v2f IN):SV_TARGET
            {
  				return (tex2D(_MainTex,IN.texcoord) + _TextureSampleAdd) * IN.color;
			}
            ENDCG
		}
	}
}
```

### 4.6 UI Canvas Rebuild

Canvas rebuild造成性能问题的两个主要原因：

* 如果在一个Canvas中绘制大量的UI元素，那么计算这些batch将是非常昂贵的。只是因为UI元素的的排序与分析的开销与其数量是超过线性变化的。
* 如果canvas仅有少量的变动却频繁的被标记为dirty，那么将花费大量的时间来refresh。

以上两个问题会随UI元素的数量的增加将变得越来越严重。

注意：任何Canvas上可绘制的UI元素的变化，该Canvas都必须重新运行rebuild过程。在这个过程中会re-analyzes这个Canvas上的每个UI元素，无论这个UI元素有没变化。这里说的“change”包括sprite的改变，transform位置和缩放的改变以及text组件的mesh改变等等。

### 4.7 Child order

UGUI是back-to-front构造的，object在层次视图中的顺序决定了它们的渲染顺序。它们的depth越深，越晚渲染。Batch是从层次视图的top-to-bottom构建的并且在这个过程中会收集使用相同材质的所有对象。使用相同texture的UI如果被插入中间层，会被强制破坏batch。利用Unity Frame Debugger来查看哪些UI被中间层破坏。这个问题通常会发生在text和sprite之间。

解决办法：

* 调整text和sprite的顺序，使同batch之间不会有其他batch插入；
* 调整object的位置消除不可见重叠空间；

### 4.8 Splitting Canvases

Sibling Canvases通常用于将部分UI元素与整个UI元素隔离。这样可以是这部分UI可以在整体之上或者之下。并且sub-canvas从parent-canvas继承了渲染信息。但是我们必须记住不同canvas之间是不能够合并到同一个batch中的。所以splitting canvas必须考虑最小的rebuild消耗和最小的drawcall的平衡。

### 4.9 General guidelines

由于可绘制对象的组件的改变会引起canvas的rebatch，所以需要将不会改变的UI元素和变动的UI元素分割到不同的sub-canvas中，最好将那些同时发生变化的分割到同一个sub-canvas，这样将避免增加太多的drawcall。

### 4.10 Unity5.2 and Optimized Batching

在Unity5.2中，batch的代码已经被重写。在多核设备上，Unity的UI系统将把大量工作迁移到工作线程中完成。在Unity5.2中减少了将一个UI分割到多个sub-canvas中的侵害。在移动设备上大部分少于2到3个canvas UI有比较好的性能。[detail info](http://blogs.unity3d.com/2015/09/07/making-the-ui-backend-faster/) 

### 4.11 Input and Raycasting in Unity UI

默认，Unity UI使用Graphic Raycaster组件来控制输入事件。这些事件通常由Standalone Input Manager组件来处理，这是一个统一的输入系统，能够处理鼠标指针和触碰事件。

#### 4.11.1 Erroneous mouse input detection on mobile(5.3)

在Unity5.4之前，只要没有touch输入可用，每个带有Graphic Raycaster组件的canvas每帧都将执行一次raycaster检测鼠标指针位置，并且这个检测是无关平台的。这是非常浪费CPU性能的，大概会消耗5%甚至更多CPU time。在Unity5.4这个问题会被修复，并且在移动设备上讲不在查询鼠标的位置也不会执行不必要的raycaster操作。如果使用的Unity5.4之前的版本，可以通过注释standalone input Manager中的所有调用`ProcessMouseEvent`的方法。

#### 4.11.2 Recast optimization

UGUI的GraphicRaycaster的实现是简单粗暴的遍历所有的Graphic组件，如果这个Graphic组件的RaycastTarget通过所有的测试，将会被加入到hit列表中。

测试过程：

* 如果RaycastTarget是active，enabled并且drawn(has geometry);
* 如果输入点在RaycastTarget的RectTransfrom范围内；
* 如果RaycastTarget有或者其子节点有任何实现了ICanvasRaycastFilter接口的组件，并且RaycastFilter组件允许Raycast。

RaycastTarget的hit列表经过depth排序，过滤reversed target，并且过滤camera之后渲染的UI元素(off-screen UI)。

如果设置Graphic Raycaster的BlockingObjects属性为各自的flag，那么Graphic Raycaster可能抛出射线到3D或者2D物理系统。如果2D或者3D的blocking object属性被启用，那么任意RaycastTarget将被绘制在2D或者3D物体的raycast-blocking Physics Layer，此时可能会排出在hit列表中。

优化建议：

1. 让所有的Recast Traget必须通过Graphic Raycaster的测试，仅仅启用‘Recast Traget’必须接收鼠标事件的UI组件。
2. 对于复合的UI(有多个可绘制UI对象)，仅仅只在复合UI的最外层加上一个Raycast Traget。
3. Hierarchy depth和Raycast Filter：使用CanvasGroup，ImageMask，RectMask2D，减少不必要的消耗；
4. Sub-canvases和overrideSorting。

##  05 Optimizing UI Controls

### 5.1 UI Text

Unity内置的Text组件是一种方便显示栅格化的文本的控件。当增加一个文本到Text组件时，每一个字符都是独立的四边形，这些四边形的glyph会占用周围的一些空间，这样不经意间会破坏其他UI元素的batch。

#### 5.1.1 Text Mesh rebuilds

TextMesh的最大问题是无论Text何时改变，它必须重新计算用于显示text的多边形，并且当它自身或者其parent节点disable或者re-enabled时，即使text没有改变，也会再次重计算多边形。当UI存在大量的Label的元素时，频繁的hide和show将造成很大的帧率降低。

#### 5.1.2 Dynamic fonts and font atlases

### 5.2 Scroll Views

UGUI的Scroll Views是造成UI性能问题的主要因素。ScrollView通常会创建大量的UI元素来显示内容。处理ScrollView的两个办法：

* 只填充ScrollView显示区域的UI元素，通常循环利用这些UI元素来达到显示全部内容；
* UI元素对象池，重定位UI元素的显示位置；
* 添加RectMask2D组件，能够确保不再ScrollView显示区域的元素不会添加到重建canvas的geometry，sorted，analyzed的列表中；

#### 5.2.1 Simple Scroll View element pooling



## 06 Other UI Optimization Techniques and Tips

### 6.1 RectTransform-Based Layouts

当Layout组件重计算其子节点元素的大小和位置时，消耗是非常大的。如果只是一个比较小的数量固定的UI元素，并且结构比较简单，我们可以采用RectTransfrom-Based布局替代。

通过给RectTransform的anchors赋值，该RectTransfrom的位置和大小能够基于它的父节点缩放。这个计算过程是native Transform System执行的，效率比较高。

### 6.2 Disabling Canvas Renderers

通过需要show或者hide部分UI时，我们一般采用enable或者disable其UI元素的root结点，确保ui不再接受输入或者Unity的callback函数。但是这将造成Canvas丢失VBO数据，重新enable这个Canvas时将再次请求Canvas以及其sub-Canvas再次执行rebuild以及rebatch的过程。如果它频繁的发生，将增加CPU usage降低frame rate。

一种变通的方案是show/hide UI元素的Canvas或者Sub-Canvas并且仅仅enable/disable Canvas或者Sub-Canvas上附着的CanvasRenderer组件。该方案不会重新绘制UI的mesh而且在内存中保留它们数据以及原始batch，但是OnEnable或者OnDisable回调将不再生效。

无论何种方案都无法排除来自于GraphicRegistry中的UI‘s Graphics，所以仍然存在于检测Graphic Raycast的组件列表中。不会disable这个hide的UI上任何MonoBehaviour脚本行为，所以这些MonoBehaviour将仍然接受Unity的生命周期的回调函数。为了避免这个问题，我们应该禁用这些脚本的行为，通过一个统一的UIManager来处理这些回调函数的发成。

### 6.3 Assigning Event Cameras

如果使用Unity内置的InputManager并设置Canvas的渲染为WorldSpace或者ScreenSpace-Camera模式，那么设置Event Camera或者Render Camera的属性是非常重要的。如果这个属性没有设置，那么Unity的UI将搜索场景中的tag为MainCamera标记的GameObject。而这个查询过程是非常耗时的，所以我们需要自己设置worldCamera等属性。
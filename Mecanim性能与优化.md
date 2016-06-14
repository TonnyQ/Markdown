# Mecanim性能与优化

## 0x001 Character Setup

+ Number of Bones : 在某种情况下，可能需要创建大量骨骼数的角色，例如当我们想要大量的自定义挂点时。这些额外的骨骼将增加模型的包的大小，并且每一个额外的骨骼都将产生一个额外的性能消耗。例如，在一个已经有30根骨骼的模型上附加额外的15根骨骼在Generic模式下将带来超过50%的消耗。注意，在Generic和Humanoid模式下都能够附加额外的骨骼。然而，当我们不播放动画时附加的额外骨骼产生的性能消耗是微不足道的。如果它们附着对象不存在或者是隐藏状态，产生的消耗将更低。
+ Mutilple Skinned Meshes : 无论何时尽可能的合并SkinnedMeshRenderer。将一个角色分成两个SkinnedMeshRenderer在性能是一个糟糕的想法。最佳的选择是一个角色使用一个材质，但是通常我们都会需要对各材质。例如，在角色换装时，我们可能将人物的Mesh和材质分开，合并也只合并SkinnedMeshRenderer。

## 0x002 Animation System

+ Controllers : 当没有设置Controller组件时，Animator组件不会产生太大的开销。
+ Simple Animation : 比起legacy animation system，mecanim animation system在播放无blending的单个animationclip时要慢。旧动画系统是非常直接的采样动画曲线，然后直接写入transform。Mecanim有一个临时的缓冲区，用于动画混合，在采样曲线和其他数据时有一个额外的拷贝。Mecanim的设计是为了优化动作混合以及更复杂动画的创建。
+ Scale Curves : Animation scale curves比较animation translation和rotation曲线有更大的开销，为了提高性能，避免使用scale animation.注意，如果是常数曲线则不受影响，常数曲线是被优化过的，比起常规曲线，常数曲线有更小的开销。
+ Layers : 大部分时间是评价Mecanim系统动画，和animationlayers和animationstatemachines开销保持在最低限度。为动画添加另一个层，同步与否的开销，取决于动画和混合树在哪一层播放。当该层的权重为零时，该层更新将被跳过。

## 0x003 Humanoid vs Generic Modes

+ 如果IK Goals和fingers animation不需要，在导入Humanoid动画时使用BodyMask来移除它们。
+ 当使用Generic模式时，使用root motion比起不是时有更大的开销，如果你的动画不使用root motion，请确保你没有选择root bone。

## 0x004 Mecanim Scene

1. 使用Hash值替代String来查询Animator
2. 实现一个小的AI Layer来控制Animator，你能使得它提供例如OnStateChange,OnTransitionBegin等等简单的回调函数
3. 使用State Tags来简单匹配你的AI State Machine对于Mecanim state machine
4. 使用附加曲线来模拟事件
5. 使用附件曲线来标记动画，例如target matching中的结合

## 0x005 Runtime Optimizations

### Visibility and Updates

+ 基于Renderer设置animator的Culling Mode来优化animations

+ 当角色Offscreen时，禁用SkinnedMeshRenderer的Update更新

+ 当角色不可见时，禁用SkinnedMeshRenderer的Update更新

  ​

  ​
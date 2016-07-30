# Best Particle==>LoadingScreenBar

## Example1:

```csharp
private IEnumerator StartLoading(int scene)
{
	AsyncOperation op = SceneManager.LoadSceneAsync(scene);
  	while(!op.isDone){
      StartLoadingPercentage(op.progress * 100);
      yield return new WaitForEndOfFrame();
  	}
}
```

问题：进度条并没有连续的显示加载的进度，而是停顿一下切换一个数字，并且最后也不会显示100%完成，而是直接切换到主场景。其根本原因在于`LoadSceneAsync`并不是真实的后台进程，在每一帧加载一些资源，并给出一个progress值，所以加载时候会造成游戏卡顿。

## Example2：

```c#
private IEnumerator StartLoading(int scene){
  AsyncOperation op = SceneManager.LoadSceneAsync(scene);
  op.allowSceneActivation = false;
  while(!op.isDone){
    SetLoadingPercentage(op.progress * 100);
    yield return new WaitForEndOfFrame();
  }
  op.allowSceneActivation = true;
}
```

问题：将AsyncOperation.allowSceneActivation=false,加载完成后设置为true。当执行时，进度条最后会一直停留在90%上，场景不会切换。原因是AsyncOperation.isDone=false,AsycnOperation.progress的值增加到0.9后就保持不变，即场景永远不会被加载完毕。其根本原因是把allowSceneActivation设置为false，Unity就只会加载场景到90%，剩余的10%要等到allowSceneActivation设置为true后才加载。

```csharp
private IEnumerator StartLoading(int scene){
  AsyncOperation op = SceneManager.LoadSceneAsync(scene);
  op.allowSceneActivation = false;
  while(op.progress < 0.9f){
    SetLoadingPercentage(op.progress * 100);
    yield return new WaitForEndOfFrame();
  }
  SetLoadingPercentage(100);
  yield return new WaitForEndOfFrame();
  op.allowSceneActivation = true;
}
```

## Example3:

```c#
private IEnumerator StartLoading(int scene)
{
  int displayProgress = 0;
  int toProgress = 0;
  AsyncOperation op = SceneManager.LoadSceneAsync(scene);
  op.allowSceneActivation = false;
  while(op.progress < 0.9f)
  {
    toProgress = (int)op.progress * 100;
    while(displayProgress < toProgress){
      ++displayProgress;
      SetLoadingPercentage(displayProgress);
      yield return new WaitForEndOfFrame();
    }
  }
  toProgress = 100;
  while(displayProgress < toProgress){
    ++displayProgress;
    SetLoadingPercentage(displayProgress);
    yield return new WaitForEndOfFrame();
  }
  op.allowSceneActivation = true;
}
```

解决方案：如果在加载游戏主场景之前还需要解析数据表格，生成对象池，进行网络连接等操作，那么可以给这些操作赋予一个权重，利用这些全职就可以计算加载的进度。或者如果场景加载的非常快，那么可以使用一个假的进度条模拟加载loading的过程，然后在加载场景。


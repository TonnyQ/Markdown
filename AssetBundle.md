# AssetBundle
AssetBundles can contain any kind of asset type recognized by Unity, as determined by the filename extension. If you want to include files with custom binary data, they should have the extension “.bytes”. Unity will import these files as TextAssets.

### 0x001 AssetBundle Workflow
1. Building AssetBundles. Asset bundles are created in the editor from assets in your scene.
2. Uploading AssetBundles to external storage.
3. Downloading AssetBundles at runtime from your application.
4. Loading objects fomr AssetBundles.

### 0x002 Building AssetBundles
At the very bottom of the inspector window for that asset,Clicking this will reveal the names of any currently defined asset bundles.plus the option to define a new bundle.the same assetbundle can contains other assets,only need give same assetbundle name for other assets.The meta file belonging to an asset will have the chosen AssetBundle name wirtten into it.

#### Exporting AssetBundles
Example:
```csharp
using UnityEditor;
public class CreateAssetBundles
{
    [MenuItem ("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles ()
    {
        //creates the AssetBundles that have beem labeled.ecah AssetBundle has an associated file a ".mainfest" extension,it provies the CRC and asset dependenices info.
        BuildPipeline.BuildAssetBundles ("Assets/AssetBundles", BuildAssetBundleOptions.None, BuildTarget.StandaloneOSXUniversal);
    }
}
```

#### Shader stripping
When include shaders in bundle,the Unity editor looks at the current scene and lightmapping setting to decide which `Lightmap` modes to use.This means that you need to have a configured scene open when building the bundle.however,you can manually specify which scene to calculate `Lightmap` modes from.This is necessary when building bundles from the commandline.Open the scene in the **Graphics Setting inspector(Editor->Project Setting->Graphics)**,go to **Shader stripping/Lightmap modes** and select **Manual**,then select **From current scene**.

#### AssetBundle editor tools
1. Getting names of AssetBundles,use `AssetDatabse.GetAllAssetBundleNames()`.
2. Getting told when an asset changes AssetBundle,use the `OnPostprocessAssetbundleNameChanged()` method from the `AssetPostprocessor` class to get callback when an asset from the AssetBundle.

#### AssetBundle variants
The Unity build pipeline gives the objects in these two variant `AssetBundle` the same internal IDs.The full AssetBundle name is the combination of the AssetBundle name and the variant name.
1. From the Editor,use the one extra variant name to the right of the asset labels GUI.
2. In code,use the `AssetImport.assetBundleVariant` option.

#### Scriping advice
1. **API to mark the asset as an AssetBundle**,use `AssetImport.assetBundleName` to set the AssetBundle name.
2. **Building AssetBundles**,use `BuildPipeline.BuildAssetBundles()`,parameters contains *output path for all the AssetBundles*,`BuildAssetBundleOptions`,`BuildTarget`,An oveeloaded version to provide an array of `AssetBundleBuild` which contains one map from assets to AssetBundles.This provies flexibility to you,you can set your mapping information by script and build from it.This mapping information does not replace or break the existing one in the asset database.
3. **API to mainpulate AssetBundles names in tha asset database**,use `AssetDatabse.GetAllAssetBundleNames()` returns all the AssetBundle names;`AssetDatabse.GetAssetPathsFromAssetBundle` return the asset paths marked in the given AssetBundle;use `AssetDatabse.RemoveAssetBundleName()` removes a given AssetBundle name in the asset database;use `AssetDatabse.GetUnusedAssetBundleNames()` returns the unused AssetBundle names;use `AssetDatabse.RemoveUnusedAssetBundleNames()` removes all the unused AssetBundel names in the asset database;use the callback `AssetPostprocessor.OnPostprocessAssetbundleNameChanged()` is called if user changes the AssetBundle name of an asset.
4. **BuildAssetBundleOptions**,`CollectDependcecies` and `DeterministicAssetBundle` are always enabled;`CompleteAssets` is ignored as it always starts from rather than objects;`ForceRebuildAssetBundle` Even if there is no change to the assets,you can force rebuild the AssetBunle by setting this flag.`IngoreTypeTreeChanges`,Even if the typetree changes,you can ignore the changes with this flag.`DisableWriteTypeTree` conflicts with `IngoreTypeTreeChanges`,you cannot ignore typetree change if you disable typetree.
5. **Single mainfest assetBundle**,only contains an `AssetBundleManifest` object.use `GetAllAssetBundles()` return all the AssetBundle names in this build.use `GetDirectDependencies()` return the direct dependent AssetBundle names,use `GetAllDependencies()` return all the dependent AssetBundle names,use `GetAssetBundleHash(string)` return the hash for the specified AssetBundle,use `GetAllAssetBundlesWithVariant()` returns all the AssetBundles with variant.
6. **AssetBundle loading API**,use `AssetBundle.GetAllAssetBundleNames()`,return all the asset names in the AssetBundle;use `AssetBundle.GetAllScenePaths()`,return all the scene asset paths if it's a streamed scene AssetBundle;use `AssetBundle.LoadAsset()`,Loads assets from the AssetBundle;use `AssetBundle.LoadAllAssets()`,load all asset;use `AssetBUndle.LoadAssetWithSubAssets()`

### 0x003 Asset Bundle Compression
Unity supports three compression options for Asset Bundles:LZMA,LZ4,and Uncompressed.
1. **LZMA**
:By default,The standard compressed format is a single LZMA stream of serialized data files,and need to be decompressed in its entirety before use.LZMA bundles give the **smallest** possible download size,but has relatively **slow compression** resulting in higher apparent load times.

2. **LZ4**
:use LZ4 compression,which results in **larger compressed file size**,but does **not require the entire bundle to decompressed before use**.LZ4 is a "chunk-based" algorithm,and therefore when objects are loaded from LZ4-compressed bundle,only the corresponding chunks for that object are decompressed,this occurs on-the-fly(runtime).The LZ4 was introduced in **Unity 5.3**.

3. **Uncompressed**
:The third compression option is not compression at all.Uncompressed bundle are **largest**,but are the **faster** to access once downloaded.

### 0x004 Caching of Compressed Bundles
The **WWW.LoadFromCacheOrDownload** function downloads and caches asset bundles to disk and thus greatly speeds up loading afterwards.From Unity5.3 onwards,cached data can also be compressed with LZ4 algorithm.This saves **40%-60%** of space compared to Uncompressed bundles.**Recompression** happens during download.As data arrives from the socket,Unity will decompress it and recompress it in the LZ4.The Recompression occurs download is complete.After that,data is read from the cached bundle by decompressing chunks on-the-fly when needed.Finally,Cache compression is enabled by default and is controlled by the **Caching.compressionEnabled** property.it affects bundles cached to disk and stored in memory.

### 0x005 AssetBundle load API overview
| Compression Format | Uncompressed | Chunk Compressed(LZ4) | Stream Compressed(LZMA) |
|--------------------|--------------|-----------------------|-------------------------|
| www* | Mem:uncompressed bundle size+(while WWW is not disposed,uncompressed bundle size).Perf:not extra processing. | Mem:LZ4HC compressed bundle size + (while WWW is not disposed,LZ4HC bundle size).Perf:no extra processing. | Mem:LZ4 compressed bundls size+(while WWW is not disposed,LZMA bundle size).Perf:LZMA decompression + LZ4 compression during download. |
| LoadFromCacheOrDownload | Mem:no extra memory is used.Perf:reading from disk. | Mem:no extra memory is used.Perf:reading from disk. | Mem:no extra memory is used.Perf:reading from disk. |
| LoadFromMemory(Async) | Mem:uncompressed bundle size.Perf:no extra processing. | Mem:LZ4HC compressed bundle size.Perf:no extra processing. | Mem:LZ4 compressed bundle size.Perf:LZMA decompression + LZ4 compression. |
| LoadFromFile(Async) | Mem:no extra memory is used.Perf:reading from disk. | Mem:no extra memory is used.Perf:reading from disk. | Mem:LZ4 compressed bundle size.Perf:reading from disk + LZMA decompression + LZ4 compression. |   
| WebRequest(supports caching) | Mem:Uncompressed bundle size.Pref:no extra processing | Mem:LZ4HC compressed bundle size.Pref: no extra proecssing | Mem:LZ4 compressed bundle size.Perf:LZMA decompression + LZ4 compression during download |
* When downloading a bundle using WWW,WebRequest there is also an 8x64KB accumulator buffer which stores data from a socket.
* Generally always choose asynchronous function-they don't stall the main thread and allow loading operations to be queued more efficiently.and absolutely avoid calling calling synchronous and asynchronous function at the same time-this might introduce hiccups on the main thread.

### 0x006 Use guidelines when using low-level loading API
1. Deploying asset bundle with your game as **StreamingAssets**-Use **BuildAssetBundleOptions.ChunkBasedCompression** when building bundles and **AssetBundle.LoadFromFileAsync** to load it.This gives your data compression and the faster possible loading performance with a memory overhead equal to read buffers.
2. Downloading asset bundles as DLCs-use **default build options(LZMA compression)** and **LoadFromCacheOrDownload/WebRequest** to download and cache it.This have the best possible compression ratio and **AssetBundle.LoadFromFile** loading performance for further loads.
3. Encrypted bundles-choose **BuildAssetBundleOptions.ChunkBasedCompression** and use **LoadFromMemoryAsync** for loading.(this is best scenario where LoadFromMemoryAsync should be used).
4. Custom bundles-use **BuildAssetBundleOptions.UncompressedAssetBundle** to build and **AssetBundle.LoadFromFileAsync** to load a bundle after it was decompressed by your custom compression algorithm.

### 0x007 Asset Bundle Internal Structures
An AssetBundle is essentially a set of objects groups together into a serialized file.it is deployed as a data file which has a slightly different Structures depending on whether it is a normal bundle or a scene bundle.
1. Normal AssetBundle Structures
![Normal AssetBunlde Structures](http://docs.unity3d.com/uploads/Main/AssetBundleStructureNormal.png)

2. Streamed scene AssetBundle Structures
![Scene AssetBundle Structures](http://docs.unity3d.com/uploads/Main/AssetBundleStructureStreamedScene.png)

3. AssetBundle compression
![AssetBundle compression](http://docs.unity3d.com/uploads/Main/AssetBundleArchiveFileSystem.png)
The Compressed blocks shown above might have chunk-based compression or stream-based compression.**chunk-based compression(LZ4)** means that the original data is split to chunks(subblocks) of equal size and that chunks are compressed independently.You should use this if you want realtime decompression-random read overhead is small.**stream-based compression(LZMA)** uses the same dictionary when processing the whole block,it provides the highest possible compression ratio but supports only sequential reads.

### 0x008 Downloading AssetBundles
1. **Non-caching**:using a creating a new **WWW** object.The AssetBundles are not cached to Unity's Cache folder in the local storage device.
2. **Caching**:using the **WWW.LoadFromCacheOrDownload** call.The AssetBundles are cached to Unity's Cache folder in the local storage device.The WebPlayer shared cache allow up to 50M of cached AssetBundles.PC/MAC standalone application and iOS/Android application have a limit of 4GB.WebPlayer application that make use of a dedicated cache are limit to the number of bytes specified in the cacheing license agreement.
3. **Loading AB in the Editor**:using `Resources.LoadAssetAtPath`(Editor only).

Example(Non-caching):
```csharp
using System;
using UnityEngine;
using System.Collections;

class NonCachingLoadExample : MonoBehaviour {
   public string BundleURL;
   public string AssetName;
   IEnumerator Start() {
     // Download the file from the URL. It will not be saved in the Cache
     using (WWW www = new WWW(BundleURL)) {
         yield return www;
         if (www.error != null)
             throw new Exception("WWW download had an error:" + www.error);
         AssetBundle bundle = www.assetBundle;
         if (AssetName == "")
             Instantiate(bundle.mainAsset);
         else
             Instantiate(bundle.LoadAsset(AssetName));
         // Unload the AssetBundles compressed contents to conserve memory
         bundle.Unload(false);

     } // memory is freed from the web stream (www.Dispose() gets called implicitly)
   }
}
```
Example(Caching **Recommanded**):
```csharp
using System;
using UnityEngine;
using System.Collections;

public class CachingLoadExample : MonoBehaviour {
    public string BundleURL;
    public string AssetName;
    public int version;

    void Start() {
        StartCoroutine (DownloadAndCache());
    }

    IEnumerator DownloadAndCache (){
        // Wait for the Caching system to be ready
        while (!Caching.ready)
            yield return null;

        // Load the AssetBundle file from Cache if it exists with the same version or download and store it in the cache
        using(WWW www = WWW.LoadFromCacheOrDownload (BundleURL, version)){
            yield return www;
            if (www.error != null)
                throw new Exception("WWW download had an error:" + www.error);
            AssetBundle bundle = www.assetBundle;
            if (AssetName == "")
                Instantiate(bundle.mainAsset);
            else
                Instantiate(bundle.LoadAsset(AssetName));
            // Unload the AssetBundles compressed contents to conserve memory
            bundle.Unload(false);
        } // memory is freed from the web stream (www.Dispose() gets called implicitly)
    }
}
```

Example:(Load AB in Editor)
```csharp
// C# Example
// Loading an Asset from disk instead of loading from an AssetBundle
// when running in the Editor
using System.Collections;
using UnityEngine;

class LoadAssetFromAssetBundle : MonoBehaviour
{
    public Object Obj;

    public IEnumerator DownloadAssetBundle<T>(string asset, string url, int version) where T : Object {
        Obj = null;

#if UNITY_EDITOR
        Obj = Resources.LoadAssetAtPath("Assets/" + asset, typeof(T));
        yield return null;

#else
        // Wait for the Caching system to be ready
        while (!Caching.ready)
            yield return null;

        // Start the download
        using(WWW www = WWW.LoadFromCacheOrDownload (url, version)){
            yield return www;
            if (www.error != null)
                        throw new Exception("WWW download:" + www.error);
            AssetBundle assetBundle = www.assetBundle;
            Obj = assetBundle.LoadAsset(asset, typeof(T));
            // Unload the AssetBundles compressed contents to conserve memory
            bundle.Unload(false);

        } // memory is freed from the web stream (www.Dispose() gets called implicitly)
#endif
    }
}
```

* When access the `.assetBundle` property,the downloaded data is extracted and the assetBundle object is created.At this point,you are ready to load the objects contained in the bundle.The sceond parameter passed to `LoadFromCacheOrDownload` specified which version of the AssetBundle to download.if the AssetBundle doesn't exist in the cache or has a version lower than requested,`LoadFromCacheOrDownload` will download the AssetBundle.Otherwise the AssetBundle will be loaded from cache.

* Only up to one AssetBundle download can finish per frame when they are downloaded with `WWW.LoadFromCacheOrDownload`.

### 0x009 Loading and Unloading objects from an AssetBundle
1. `AssetBundle.LoadAsset()` will load an object using its name identifier as a parameter.the name is the one visible in the project view.you can optionally pass an object type as an argument to the Load method to make sure the object load is of a specific type.
2. `AssetBundle.LoadAssetAsync()` works the same as the load method described above but it will not block the main thread while the asset is loaded.This is useful when loading large assets or many assets at once to avoid pauses in app.
3. `AssetBundle.LoadAllAssets()` will load all the objects contained in AssetBundle.as with `AssetBundle.LoadAsset()`,you can optionally filter objects by their type.
4. `AssetBunle.Unload()`,this method takes a boolean parameter which tells Unity whether to unload all data(including the loaded asset objects) or only the compressed data from the download bundle.if your app is using some objects from the `AssetBunle` and you want to free some memory you can pass `false` to unload the compressed data from memory;if you want to completely unload everything from the `AssetBunle` you should pass `true` which will destroy the Asset loaded from the AssetBundle.

Example(async Loading object from AB):
```csharp
using UnityEngine;

// Note: This example does not check for errors. Please look at the example in the DownloadingAssetBundles section for more information
IEnumerator Start () {
    while (!Caching.ready)
        yield return null;

    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);
    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;
    // Load the object asynchronously
    AssetBundleRequest request = bundle.LoadAssetAsync ("myObject", typeof(GameObject));
    // Wait for completion
    yield return request;

    // Get the reference to the loaded object
    GameObject obj = request.asset as GameObject;
    // Unload the AssetBundles compressed contents to conserve memory
    bundle.Unload(false);
    // Frees the memory from the web stream
    www.Dispose();
}
```

### 0x010 Keeping Track of loaded AssetBundles
Unity will only allow to have a single instance of a particular `AssetBundle` load at one time in your application.Once load the same one has been loaded previously and has not been unloaded,will get a error and the `www.assetBunle` property will return `null`.

1. **Unload** the AssetBundle when no longer using it.(**Recommanded**,saved memory)
2. **Maintain** a reference to it and avoid downloading it.

Example:(keep track of AB):
```csharp
using UnityEngine;
using System;
using System.Collections;
using System.Collections.Generic;

static public class AssetBundleManager {
   // A dictionary to hold the AssetBundle references
   static private Dictionary<string, AssetBundleRef> dictAssetBundleRefs;
   static AssetBundleManager (){
       dictAssetBundleRefs = new Dictionary<string, AssetBundleRef>();
   }
   // Class with the AssetBundle reference, url and version
   private class AssetBundleRef {
       public AssetBundle assetBundle = null;
       public int version;
       public string url;
       public AssetBundleRef(string strUrlIn, int intVersionIn) {
           url = strUrlIn;
           version = intVersionIn;
       }
   };
   // Get an AssetBundle
   public static AssetBundle getAssetBundle (string url, int version){
       string keyName = url + version.ToString();
       AssetBundleRef abRef;
       if (dictAssetBundleRefs.TryGetValue(keyName, out abRef))
           return abRef.assetBundle;
       else
           return null;
   }

   // Download an AssetBundle
   public static IEnumerator downloadAssetBundle (string url, int version){
       string keyName = url + version.ToString();
       if (dictAssetBundleRefs.ContainsKey(keyName))
           yield return null;
       else {
           while (!Caching.ready)
               yield return null;

           using(WWW www = WWW.LoadFromCacheOrDownload (url, version)){
               yield return www;
               if (www.error != null)
                   throw new Exception("WWW download:" + www.error);
               AssetBundleRef abRef = new AssetBundleRef (url, version);
               abRef.assetBundle = www.assetBundle;
               dictAssetBundleRefs.Add (keyName, abRef);
           }
       }
   }
   // Unload an AssetBundle
   public static void Unload (string url, int version, bool allObjects){
       string keyName = url + version.ToString();
       AssetBundleRef abRef;
       if (dictAssetBundleRefs.TryGetValue(keyName, out abRef)){
           abRef.assetBundle.Unload (allObjects);
           abRef.assetBundle = null;
           dictAssetBundleRefs.Remove(keyName);
       }
   }
}
```

### 0x011 Storing and loading binary data in an AssetBundle
1. The first step is to save your binary data file with the `.bytes` extension.
2. Unity will treat this file as a `TextAssets`,As a TextAssets the file can be include when you build your AssetBundle.
3. Download the AssetBundle and loaded the TextAsset object,can use the `.bytes` property of the `TextAsset` to retrieve binary data.

Example:
```csharp
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    while (!Caching.ready)
        yield return null;

    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);
    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;

    // Load the TextAsset object
    TextAsset txt = bundle.Load("myBinaryAsText", typeof(TextAsset)) as TextAsset;
    // Retrieve the binary data as an array of bytes
    byte[] bytes = txt.bytes;
}
```

### 0x012 Protecting contents
1. making use of the `TextAsset` type to store data as bytes.Encrypt data file and save them with a `.bytes` extension.the AssetBundle would be downloaded and the content decrypted from the bytes stored in the TextAsset.with this method the assetBundle are not encrypted,but the data stored which is stored as TextAsset is.
2. Encrypt the `AssetBundle` from source and then download them using the `WWW` class.Once downloaded would use your decryption routine on the data from the `.bytes` property of WWW instance to get the decrypted AssetBundle file data and create the AssetBundle from memory using `AssetBundle.CreateFromMemory()`
3. Store an AssetBundle itself as a `TextAsset`,inside another normal `AssetBundle`.the unencrypted `AssetBundle` containing the entrypt one would be cached.The original `AssetBundle` could be loaded into memory,decrypted and Instantiate using `AssetBundle.CreateFromMemory()`.

Example1:
```csharp
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    while (!Caching.ready)
        yield return null;

    // Start a download of the encrypted assetbundle
    WWW www = new WWW.LoadFromCacheOrDownload (url, 1);
    // Wait for download to complete
    yield return www;

    // Load the TextAsset from the AssetBundle
    TextAsset textAsset = www.assetBundle.Load("EncryptedData", typeof(TextAsset));
    // Get the byte data
    byte[] encryptedData = textAsset.bytes;
    // Decrypt the AssetBundle data
    byte[] decryptedData = YourDecryptionMethod(encryptedData);
    // Use your byte array. The AssetBundle will be cached
}
```

Example2:
```csharp
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    // Start a download of the encrypted assetbundle
    WWW www = new WWW (url);
    // Wait for download to complete
    yield return www;

    // Get the byte data
    byte[] encryptedData = www.bytes;
    // Decrypt the AssetBundle data
    byte[] decryptedData = YourDecryptionMethod(encryptedData);

    // Create an AssetBundle from the bytes array
    AssetBundleCreateRequest acr = AssetBundle.CreateFromMemory(decryptedData);
    yield return acr;

    AssetBundle bundle = acr.assetBundle;
    // You can now use your AssetBundle. The AssetBundle is not cached.
}
```

Example3:
```csharp
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    while (!Caching.ready)
        yield return null;

    // Start a download of the encrypted assetbundle
    WWW www = new WWW.LoadFromCacheOrDownload (url, 1);
    // Wait for download to complete
    yield return www;

    // Load the TextAsset from the AssetBundle
    TextAsset textAsset = www.assetBundle.Load("EncryptedData", typeof(TextAsset));
    // Get the byte data
    byte[] encryptedData = textAsset.bytes;
    // Decrypt the AssetBundle data
    byte[] decryptedData = YourDecryptionMethod(encryptedData);

    // Create an AssetBundle from the bytes array
    AssetBundleCreateRequest acr = AssetBundle.CreateFromMemory(decryptedData);
    yield return acr;

    AssetBundle bundle = acr.assetBundle;
    // You can now use your AssetBundle. The wrapper AssetBundle is cached
}
```

### 0x013 including scripts in AssetBundle
`AssetBundle` can contain scripts as TextAsset but as such they will not be actual executable code.
1. pre-compiled into an assembly
2. loaded using the Mono Reflection class(Note:Reflection is not available on platforms that use AOT compilation,such as iOS)

Example(load scripts from AB):
```csharp
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    while (!Caching.ready)
        yield return null;

    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);
    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;
    // Load the TextAsset object
    TextAsset txt = bundle.Load("myBinaryAsText", typeof(TextAsset)) as TextAsset;

    // Load the assembly and get a type (class) from it
    var assembly = System.Reflection.Assembly.Load(txt.bytes);
    var type = assembly.GetType("MyClassDerivedFromMonoBehaviour");

    // Instantiate a GameObject and add a component with the loaded class
    GameObject go = new GameObject();
    go.AddComponent(type);
}
```

### 0x014 AssetBundles FAQ
1. If your AssetBundles are stored in the **StreamingAssets** folder as Uncompressed AssetBundles,you can use `AssetBundle.CreateFromFile()` to reference the AssetBundle on disk.if the AssetBundle in the **StreamingAssets** folder are compressed,you will need to use `WWW.LoadFromCacheOrDownload` to create an Uncompressed copy of the AssetBundle in cache.

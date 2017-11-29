## Fresco源码解析

### 解码策略
5.0以上版本使用ArtDecoder，4.4以下使用GingerbreadPurgeableDecoder，4.4到5.0之间使用KitKatPurgeableDecoder。(GingerbreadPurgeableDecoder与KitKatPurgeableDecoder为DalvikPurgeableDecoder子类)

```
  public static PlatformDecoder buildPlatformDecoder(
      PoolFactory poolFactory,
      boolean directWebpDirectDecodingEnabled) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
      int maxNumThreads = poolFactory.getFlexByteArrayPoolMaxNumThreads();
      return new ArtDecoder(
          poolFactory.getBitmapPool(),
          maxNumThreads,
          new Pools.SynchronizedPool<>(maxNumThreads));
    } else {
      if (directWebpDirectDecodingEnabled
          && Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
        return new GingerbreadPurgeableDecoder();
      } else {
        return new KitKatPurgeableDecoder(poolFactory.getFlexByteArrayPool());
      }
    }
  }
```
	
### 内存简述
基本原理参照google [BITMAP内存管理](https://developer.android.com/topic/performance/graphics/manage-memory.html)
	
以下为fresco内存池获取内存策略
	
```
public V get(int size) {
    ...
    //依据不同类型Bucket池获取Bucket大小
    int bucketedSize = getBucketedSize(size);
    int sizeInBytes = -1;
    synchronized (this) {
     //获取Bucket池中符合需要bucketSize的Bucket
      Bucket<V> bucket = getBucket(bucketedSize);
      if (bucket != null) {
        //从Bucket池中找到可以复用的内存返回使用
        V value = bucket.get();
        ...
        return value;
        }
        // fall through
      }
    ...
    V value = null;
    try {
      ...
      //如果Bucket池中不存在可复用的内存，则申请这段内存
      value = alloc(bucketedSize);
    }...
    ...
    return value;
  }
```

 * 5.0及以上系统使用ArtDecoder，利用系统inBitmap方式重用内存。
 KITKAT版本之后，只要创建的Bitmap总字节数大于需要的字节数，则可用于传递给inBitmap重用内存。
 
	 解码流程
	 
	 ```
	protected CloseableReference<Bitmap> decodeStaticImageFromStream(
	      InputStream inputStream, BitmapFactory.Options options, @Nullable Rect regionToDecode) {
	    ...
	    //从Bitmap内存池中获取可用Bitmap内存
	    final Bitmap bitmapToReuse = mBitmapPool.get(sizeInBytes);
	    ...
	    options.inBitmap = bitmapToReuse;
	    ...
	    try {
	     ...
	     //解码流获取对应Bitmap存入设置到inBitmap的内存中
	      if (decodedBitmap == null) {
	        decodedBitmap = BitmapFactory.decodeStream(inputStream, null, options);
	      }
	    } ...
	    return CloseableReference.of(decodedBitmap, mBitmapPool);
	  }
	 ```
		 
	 申请Bitmap内存空间方法
	 
	```
	@Override
	protected Bitmap alloc(int size) {
	return Bitmap.createBitmap(
	    1,
	    (int) Math.ceil(size / (double) BitmapUtil.RGB_565_BYTES_PER_PIXEL),
	    Bitmap.Config.RGB_565);
	}
	```

* 5.0以下系统使用DalvikPurgeableDecoder，利用Option的inPurgeable参数，当inPurgeable为true,会将图片解码数据存放在Ashmem内存中。Ashmem在unpin之前，将不会被Android系统回收，在unpin之后，系统内存不足时，将会被系统回收。KitKatPurgeableDecoder与GingerbreadPurgeableDecoder最大的不同之处在于源编码数据的存放位置不同。

	GingerbreadPurgeableDecoder源数据存放于inPurgeable如下可以看到，当bitmap大于32K的时候，会将解码数据存放于ashmem中。
		
	```
	BitmapFactory.cpp
	156  static SkPixelRef* installPixelRef(SkBitmap* bitmap, SkStream* stream,
	157                                   int sampleSize, bool ditherImage) {
	158    SkImageRef* pr;
	159    // only use ashmem for large images, since mmaps come at a price
	160    if (bitmap->getSize() >= 32 * 1024) {
	161        pr = new SkImageRef_ashmem(stream, bitmap->config(), sampleSize);
	162    } else {
	163        pr = new SkImageRef_GlobalPool(stream, bitmap->config(), sampleSize);
	164    }
	165    pr->setDitherImage(ditherImage);
	166    bitmap->setPixelRef(pr)->unref();
	167    pr->isOpaque(bitmap);
	168    return pr;
	169    }
			...
	290    if (isPurgeable) {
	291        pr = installPixelRef(bitmap, stream, sampleSize, doDither);
	292    }
		
	```

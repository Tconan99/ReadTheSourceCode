# [Kingfisher笔记](https://github.com/onevcat/Kingfisher.git)



## 一、知识点
1. NSCache
2. FileManager
3. DispatchQueue
4. ImageCost (静图和动图有区别)
5. NotificationCenter.default.post() // 通知中心 UIApplicationDidEnterBackgroundNotification
6. Dictionary的排序写法
7. CGImageSourceCreateWithData
8. CGImageSource
9. 单例
10. NSError
11. extension
12. 自定义运算符
13. 面对回收的原理
14. CADisplayLink
15. RunLoopMode

## 二、示例

### 1. FileManager

```
fileprivate func travelCachedFiles(onlyForCacheSize: Bool) -> (urlsToDelete: [URL], diskCacheSize: UInt, cachedFiles: [URL: URLResourceValues]) {
        
	let diskCacheURL = URL(fileURLWithPath: diskCachePath)
   let resourceKeys: Set<URLResourceKey> = [.isDirectoryKey, .contentAccessDateKey, .totalFileAllocatedSizeKey]
   let expiredDate: Date? = (maxCachePeriodInSecond < 0) ? nil : Date(timeIntervalSinceNow: -maxCachePeriodInSecond)
    
   var cachedFiles = [URL: URLResourceValues]()
   var urlsToDelete = [URL]()
   var diskCacheSize: UInt = 0

	for fileUrl in (try? fileManager.contentsOfDirectory(at: diskCacheURL, includingPropertiesForKeys: Array(resourceKeys), options: .skipsHiddenFiles)) ?? [] {

		do {
			let resourceValues = try fileUrl.resourceValues(forKeys: resourceKeys)
			// If it is a Directory. Continue to next file URL.
			if resourceValues.isDirectory == true {
				continue
			}
			
			// If this file is expired, add it to URLsToDelete
			if !onlyForCacheSize,
			    let expiredDate = expiredDate,
			    let lastAccessData = resourceValues.contentAccessDate,
			    (lastAccessData as NSDate).laterDate(expiredDate) == expiredDate
			{
			    urlsToDelete.append(fileUrl)
			    continue
			}
			
			if let fileSize = resourceValues.totalFileAllocatedSize {
			    diskCacheSize += UInt(fileSize)
			    if !onlyForCacheSize {
			        cachedFiles[fileUrl] = resourceValues
			    }
			}
		} catch _ { }
	}

	return (urlsToDelete, diskCacheSize, cachedFiles)
}
```

### 2. 排序

```
extension Dictionary {
    func keysSortedByValue(_ isOrderedBefore: (Value, Value) -> Bool) -> [Key] {
    	 // $0.1 表示leftValue.URLResourceValues
    	 // sorted 的返回值就是 (URL, URLResourceValues)
    	 // map 的 $0.0 就是 URL
        return Array(self).sorted{ isOrderedBefore($0.1, $1.1) }.map{ $0.0 }
    }
}

// cachedFiles: [URL: URLResourceValues]
// dictionary 根据Value的属性排序
// Sort files by last modify date. We want to clean from the oldest files.
let sortedFiles = cachedFiles.keysSortedByValue {
    resourceValue1, resourceValue2 -> Bool in
    
    if let date1 = resourceValue1.contentAccessDate,
       let date2 = resourceValue2.contentAccessDate
    {
        return date1.compare(date2) == .orderedAscending // 升序
    }
    
    // 很好的注释方式
    // Not valid date information. This should not happen. Just in case.
    return true
}
```

### CGImageSource

```
let options: NSDictionary = [kCGImageSourceShouldCache as String: true, kCGImageSourceTypeIdentifierHint as String: kUTTypeGIF]
guard let imageSource = CGImageSourceCreateWithData(data as CFData, options) else { return nil }

// 获取动图 frame 数量
let frameCount = CGImageSourceGetCount(imageSource)

for i in 0 ..< frameCount {
	// 获取第 i 个 frame 图片
	let imageRef = CGImageSourceCreateImageAtIndex(imageSource, i, options)
	let properties = CGImageSourceCopyPropertiesAtIndex(imageSource, i, nil)
	
	// 获取这个frame的属性
	let gifInfo = (properties as NSDictionary)[kCGImagePropertyGIFDictionary as String] as? NSDictionary
	gifDuration += frameDuration(from: gifInfo)
}


```

### 单例

static let 可以直接创建一个单例，系统会自动调用dispatch_once

### NSError

```
NSError(domain: KingfisherErrorDomain,
                                code: KingfisherError.invalidStatusCode.rawValue,
                                userInfo: [KingfisherErrorStatusCodeKey: statusCode, NSLocalizedDescriptionKey: HTTPURLResponse.localizedString(forStatusCode: statusCode)])
```


### 扩展(extension)

扩展属性(只能是计算属性)

```
public struct ImageResource: Resource {
    /// The key used in cache.
    public let cacheKey: String
    
    /// The target image URL.
    public let downloadURL: URL
    
    /**
     Create a resource.
     
     - parameter downloadURL: The target image URL.
     - parameter cacheKey:    The cache key. If `nil`, Kingfisher will use the `absoluteString` of `downloadURL` as the key.
     
     - returns: A resource.
     */
    public init(downloadURL: URL, cacheKey: String? = nil) {
        self.downloadURL = downloadURL
        self.cacheKey = cacheKey ?? downloadURL.absoluteString
    }
}

extension URL: Resource {
    public var cacheKey: String { return absoluteString }
    public var downloadURL: URL { return self }
}
```

### 自定义运算符

看起来像是炫耀技巧，这样写是否有必要

```
precedencegroup ItemComparisonPrecedence {
    associativity: none
    higherThan: LogicalConjunctionPrecedence
}

infix operator <== : ItemComparisonPrecedence

// This operator returns true if two `KingfisherOptionsInfoItem` enum is the same, without considering the associated values.
func <== (lhs: KingfisherOptionsInfoItem, rhs: KingfisherOptionsInfoItem) -> Bool {
    switch (lhs, rhs) {
    case (.targetCache(_), .targetCache(_)): return true
    case (.downloader(_), .downloader(_)): return true
    case (.transition(_), .transition(_)): return true
    case (.downloadPriority(_), .downloadPriority(_)): return true
    case (.forceRefresh, .forceRefresh): return true
    case (.forceTransition, .forceTransition): return true
    case (.cacheMemoryOnly, .cacheMemoryOnly): return true
    case (.onlyFromCache, .onlyFromCache): return true
    case (.backgroundDecode, .backgroundDecode): return true
    case (.callbackDispatchQueue(_), .callbackDispatchQueue(_)): return true
    case (.scaleFactor(_), .scaleFactor(_)): return true
    case (.preloadAllAnimationData, .preloadAllAnimationData): return true
    case (.requestModifier(_), .requestModifier(_)): return true
    case (.processor(_), .processor(_)): return true
    case (.cacheSerializer(_), .cacheSerializer(_)): return true
    case (.keepCurrentImageWhileLoading, .keepCurrentImageWhileLoading): return true
    case (.onlyLoadFirstFrame, .onlyLoadFirstFrame): return true
    case (.cacheOriginalImage, .cacheOriginalImage): return true
    default: return false
    }
}
```

### CADisplayLink 惰性加载

```
private lazy var displayLink: CADisplayLink = {
    self.isDisplayLinkInitialized = true
    let displayLink = CADisplayLink(target: TargetProxy(target: self), selector: #selector(TargetProxy.onScreenUpdate))
    displayLink.add(to: .main, forMode: self.runLoopMode)
    displayLink.isPaused = true
    return displayLink
}()
```

### RunLoopMode.commonModes

```
/// The animation timer's run loop mode. Default is `NSRunLoopCommonModes`. Set this property to `NSDefaultRunLoopMode` will make the animation pause during UIScrollView scrolling.
public var runLoopMode = RunLoopMode.commonModes {
    willSet {
        if runLoopMode == newValue {
            return
        } else {
            stopAnimating()
            displayLink.remove(from: .main, forMode: runLoopMode)
            displayLink.add(to: .main, forMode: newValue)
            startAnimating()
        }
    }
}
```

## 三、逻辑

程序员大多数时候都是直接使用这个方法来加载图片

```
// ViewController.swift

(cell as! CollectionViewCell).cellImageView.kf.setImage(with: url)
```

这个代码里面写了很多逻辑重点是加载图片这行代码

```
// ImageView+Kingfisher.swift

let task = KingfisherManager.shared.retrieveImage()
```

retrieveImage 就是加载图片的总方法, 里面包含了下载与使用缓存的判断写法

```
// KingfisherManager.swift

public func retrieveImage() {
	if options.forceRefresh {
		_ = downloadAndCacheImage()
	} else {
		tryToRetrieveImageFromCache()
	}
}
```

downloadAndCacheImage() 这个方法, 就是直接下载图片并且放在缓存里面

```
// KingfisherManager.swift

func downloadAndCacheImage() {
	downloader.downloadImage()
}
```

tryToRetrieveImageFromCache() , 这个方法是包括了 downloadAndCacheImage() 这个方法的, 如果发现缓存则加载缓存, 如果没有发现缓存则直接下载

```
// KingfisherManager.swift

func tryToRetrieveImageFromCache() {
	func handleNoCache() {
		if option.onlyFromCache {
			return
		}
		self.downloadAndCachedImage()
	}
	
	targetCache.retrieveImage() {
		if image != nil {
			completion()
			return
		}
		
		guard processor != DefaultImageProcesser.default else {
			handleNoCache()
			return
		}
		
		// 重置option, 不同的option加载缓存的结果不同
		targetCache.retrieveImage() {
			
		}
	}
}
```

retrieveImage() 主要就是从cache和disk获取数据

```
// ImageDownloader.swift

func retrieveImage() {
	if let image = self.retrieveImageInMemoryCache() {
		completion()
	} else {
		// 切换到IO线程
		self.retrieveImageInDiskCache()
	}
}
```

downloadImage() 就是下载图片, 但是涉及多线程还有 var fetchLoads = \[URL: ImageFetchLoad]() 的概念, 大意就是多处发起同一个下载, 用url做key来区分是不是同一个图片, 只要下载有进度或者有通知就通知所有挂载的位置

```
// ImageDownloader.swift

func downloadImage() {
	setup() {
		session.dataTask(with: request)	// 开始下载
	}
	
	func setup(started: _) {
		func prepareFetchLoad() {
			started()
		}
		
		prepareFetchLoad()
	}
}
```
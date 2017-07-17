# Kingfisher笔记

## 一、知识点
1. NSCache
2. FileManager
3. DispatchQueue
4. ImageCost (静图和动图有区别)
5. NotificationCenter.default.post() // 通知中心 UIApplicationDidEnterBackgroundNotification
6. Dictionary的排序写法
7. CGImageSourceCreateWithData
8. CGImageSource

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

## 2. 排序

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

## CGImageSource

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

## 单例

static let 可以直接创建一个单例，系统会自动调用dispatch_once

## NSError

```
NSError(domain: KingfisherErrorDomain,
                                code: KingfisherError.invalidStatusCode.rawValue,
                                userInfo: [KingfisherErrorStatusCodeKey: statusCode, NSLocalizedDescriptionKey: HTTPURLResponse.localizedString(forStatusCode: statusCode)])
```


## 扩展(extension)

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

## 自定义运算符

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
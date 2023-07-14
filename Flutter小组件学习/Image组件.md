# Image组件

Image小组件用来在Flutter中显示图片,包括显示4场景

* [Image.asset](#Image.asset )
* [Image.network](#Image.network) 
* [Image.file](#Image.file)
* [Image.memory](#Image.memory)
* [常用属性](#常用属性)
* [图片格式支持](#图片格式支持)

# Image.asset 

从本地asset资源中加载图片

```dart
Image.asset('assets/images/my_image.png');
```

这需要你在`pubspec.yaml`中先配置`assets`的路径

```yaml
  assets:
    - assets/images/my_image.png
```

如果图片太多了,你又不想一个一个的去配置, 也可以直接写文件夹的路径

```yaml
  assets:
    - assets/images/
```

当需要补同分辨率的图片时,你可以通过`2x`和`3x`来划分路径

```yaml
  assets:
    - assets/images/my_image.png
    - assets/images/2x/my_image2.png
    - assets/images/3x/my_image3.png
```

> 当然这看起来更加麻烦了, 我们通常的建议就是直接使用3X的图片.

# Image.network 

从指定的URL下载图片

Flutter将以加载缓存的方式显示图片

```dart
Image.network('https://example.com/my_image.png');
```

当然从网络加载的图片肯定会要请求过程, 所以如果你想在请求时显示一个loading之类的话,可以通过`loadingBuilder`来实现

```dart
Image.network(
    "https://example.com/my_image.png",
    loadingBuilder: (context, child, loadingProgress) {
        print(loadingProgress);
        return loadingProgress == null
                ? child
                : const CircularProgressIndicator();
    }
)
/*
flutter: ImageChunkEvent#1fb5d(cumulativeBytesLoaded: 13780, expectedTotalBytes: 87962)
flutter: ImageChunkEvent#0873d(cumulativeBytesLoaded: 32260, expectedTotalBytes: 87962)
flutter: ImageChunkEvent#18aaf(cumulativeBytesLoaded: 48642, expectedTotalBytes: 87962)
flutter: ImageChunkEvent#14f87(cumulativeBytesLoaded: 48644, expectedTotalBytes: 87962)
flutter: ImageChunkEvent#1c202(cumulativeBytesLoaded: 87962, expectedTotalBytes: 87962)
flutter: null
*/
```

# Image.file

从本地文件在家图像, 如手机/电脑的磁盘中

```dart
Image.file(File('path/to/my_image.png'));
```

# Image.memory

通过给定的字节数组加载图片,这个字节数组可以来自任何地方,如从网络下载或者从本地文件读取

```dart
Image.memory(bytes);
```

# 常用属性

设置图片的宽高

```dart
 Image.network(
    // URL
    'https://example.com/my_image.png',
    
    // 显示模式,如平铺,拉伸,填充等等
    fit: BoxFit.cover, 
    
    // 宽度
    width: 320,
    
    // 高度
    height: 240,
    
    // 颜色
    color: Colors.red,
    
    // 混合模式
    colorBlendMode: BlendMode.darken,
    
    // 语义标签, 用于辅助功能, 如iOS的VoiceOver之类的
    semanticLabel: "健身的图片",
)
```

# 图片格式支持


| 图片格式 | 支持 |
| --- | --- |
| JPEG | ✅ |
| PNG | ✅ |
| GIF | ✅ |
| WebP | ✅ |
| BMP | ✅ |
| WBMP | ✅ |


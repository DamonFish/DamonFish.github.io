---
layout: post
title: "压缩图片和视频资源处理"
subtitle: ""
author: "DamonFish"
header-img: "img/post-bg-infinity.jpg"
header-mask: 0.3
mathjax: true
tags:
  - 工具
  - App开发
---

## 压缩图片

图片压缩使用[TinyPNG](https://tinypng.com/cn/?ref=www.codernav.cn)，
可以下载对应的命令行工具[tinypng-cli](https://www.npmjs.com/package/tinypng-cli), 然后取官网申请一个key，一个月500次免费使用次数。
TinyPNG对web前端的童鞋再熟悉不过了，详细使用请参考官网。

以前我们想找到工程中所有的大图（比如>100k）的图片，然后压缩之，都是用脚本,类似下面：

```python
import os

Const_Image_Format = [".png", "jpeg", "pdf"]
rootDir = "/Users/You/Documents/Images.xcassets"
limit_size = 100*1024
limit_height = 100
limit_width = 100
 
class FileFilt:
    def __init__(self):
        pass
    def FilterFile(self, dirr):
        for parent,filenames in os.walk(rootDir):       
            for filename in filenames :
                fileDir = os.path.join(parent,filename)
                if fileDir and (os.path.splitext(fileDir)[1] in Const_Image_Format ):
                    filesize = os.path.getsize(fileDir)
                    if (filesize >= limit_size):
                        print(fileDir)

 
if __name__ == "__main__":
    b = FileFilt()
    b.FilterFile(dirr = rootDir)
    print('execute finished.')
```

后来发现有一个更简单的方法， find命令+tinyPNG-cli

查找某个目录下所有大于100k的文件

```
find /path/to/directory -type f -size +100k
```

查找某个目录下所有大于100k的图片文件并压缩

```
find /path/to/directory -type f -size +100k -print -exec tinypng -k key {} \;
```

查找资源目录下所有大于50k且小于100k 的 格式为 png或jpg的图片，  并压缩它们

```
find /path/to/directory -type f \( -name '*.png' -o -name '*.jpg' \) -size +50k -size -100k -print -exec tinypng -k key {} \;
```

key替换成你你申请的tinyPNG 的key， /path/to/directory 替换成你的资源文件路径

## 压缩视频

图片压缩使用[ffmpeg](https://ffmpeg.org//)

### 影响视频大小的因素

一般下面6个因素影响视频大小：

1. 视频格式format：mp4、mov、flv、avi、hls等
2. 像素格式：pix_fmt 比如 yuv420p
3. 视频编码格式codec：codec_name 如H264、H265、vp8、vp8、av1
4. 宽、高、时长：width、height、duration
5. 码率：bitrate= size*8/duration
6. 帧率：fps

### 压缩选项

压缩视频大小， 一般修改2-4项，

1. 手机播放端一般不需要这么高分辨率，降低720p；
2. 帧率30fps即可，可以降低到24左右；
3. 码率压缩，选用-crf固定码率比压缩;
4. 视频编码h264(main)可以调整到h264(high);

以上4个参数配置压缩命令如下：
```
ffmpeg -y -i xx.mp4 -c:a copy  -c:v libx264 -profile:v high -r 30 -crf 22 -s 720x1080  xx-out.mp4
```

简单介绍参数：

* -y:表示输出文件覆盖
* -i:表示输入文件
* -c:a:表示音频部分编码，copy表示直接复制到新文件
* -c:v:表示视频部分编码，libx264表示使用h264编码
* -profile:v:h264编码参数，使用更紧凑的压缩算法
* -r:表示视频帧率，30fps
* -crf:表示码率应用固定码率比，从0～500，越大码率越低，一般18～32效果较好
* -s:视频分辨率裁剪，720x1080表示裁剪成720p

你可以只修改部分参数，比如修改crf，不改变其他，效果也是立竿见影
```
ffmpeg -i input.mp4 -vcodec libx264 -crf 20 output.mp4
```

### 转换成HLS(m3u8)

在压缩完视频后， 如果是线上播放，可以考虑改为HTTP Live Streaming(m3u8)格式，减少缓冲时间：

```
ffmpeg -i input.mp4 -codec: copy -vbsf h264_mp4toannexb -start_number 0 -hls_time 10 -hls_list_size 0 -f hls output.m3u8
```

* -hls_time n: 设置每片的长度，默认值为2。单位为秒
* -hls_list_size n:设置播放列表保存的最多条目，设置为0会保存有所片信息，默认值为5
* -hls_start_number n:设置播放列表中sequence number的值为number，默认值为0

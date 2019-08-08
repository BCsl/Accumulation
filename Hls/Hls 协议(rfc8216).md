# [HLS rfc8216](https://tools.ietf.org/html/rfc8216)

了解的目的：客户端如何解析 m3u8 文件

m3u8文件有两种应用场景：多码率适配流 (Master Playlist)，单码率适配流 (Media Playlist)

一些需要了解的标签

4.3.2.4 EXT-X-KEY

媒体切片的解密方式

`#EXT-X-KEY:<attribute-list>`

属性

- METHOD，枚举，NONE, AES-128, and SAMPLE-AES
- URI, 用于标识解密密钥文件
- IV, 用于 AES 解密的向量，版本 2 以上才有
- KEYFORMAT, 版本 5 以上才有, 标识密钥在密钥文件中的存储方式, 默认是"identity"，密钥文件中的 AES-128 密钥是以二进制方式存储的 16 个字节的密钥
- KEYFORMATVERSIONS

4.3.3 Media Playlist Tags

这部分 Tags 用来描述媒体播放列表的全局信息，且不能出现在 Master Playlist 中，且在一个列表文件中只能出现最多一次

4.3.3.1 EXT-X-TARGETDURATION (必须有的)

分段信息最大的时间长的，单位（s）

`#EXT-X-TARGETDURATION:<s>`

4.3.3.2 EXT-X-MEDIA-SEQUENCE

定义当前 m3u8 文件中第一个文件的序列号，默认开始 0

`#EXT-X-MEDIA-SEQUENCE:<number>`

4.3.3.3 EXT-X-DISCONTINUITY-SEQUENCE

`#EXT-X-MEDIA-SEQUENCE:<number>`

4.3.3.4 EXT-X-ENDLIST

表示 Media Playlist 的结束

4.3.3.5 EXT-X-PLAYLIST-TYPE

`#EXT-X-PLAYLIST-TYPE:[EVENT|VOD]`

`EVENT` 直播流，还可能会添加新的媒体段（Media Segment），即 Playlist 还不完整的

4.3.3.6 EXT-X-I-FRAMES-ONLY

表示这个 Media Playlist 的每个 Media Segment 描述一个单一的 I-frame，I-frame 播放列表可用于特效播放，例如快进，快速播放、后退等

4.3.4 Master Playlist Tags

4.3.4.1 EXT-X-MEDIA

用于关联同一个内容的多个 Media Playlist 的多种 renditions（描述、角度）。例如用三个 EXT-X-MEDIA 标签标识不用音轨的音频的播放列表，有英语、法语、西班牙语等，或者用两个 EXT-X-MEDIA 标签来标识不同视觉角度拍摄的视频

`#EXT-X-MEDIA:<attribute-list>`，支持的属性 TYPE、URI、GROUP-ID、LANGUAGE、ASSOC-LANGUAGE、NAME、DEFAULT、AUTOSELECT、FORCED、INSTREAM-ID

其中 TYPE 必需，值 AUDIO, VIDEO, SUBTITLES, CLOSED-CAPTIONS

4.3.4.1.1 Rendition Groups

具有相同 GROUP-ID 和 TYPE 属性的 EXT-X-MEDIA 标签作为一组，同一组中描述的内容必须是"相同内容"的一个变种（同一视频的不同的语言版本等）

- 相同 GROUP-ID 下的 EXT-X-MEDIA 标签 NAME 属性不能相同
- 相同 GROUP-ID 下的 EXT-X-MEDIA 标签必须有一个 DEFAULT 属性为 true
- 带 AUTOSELECT=YES 属性的 EXT-X-MEDIA 标签的 LANGUAGE、ASSOC-LANGUAGE、FORCED、CHARACTERISTICS 属性组合必定不能和另外一个 AUTOSELECT=YES 属性的 EXT-X-MEDIA 标签相同

4.3.4.2 EXT-X-STREAM-INF

用于指定一个 Variant Stream（例如不同码率的同一个视频）

格式

```
#EXT-X-STREAM-INF:<attribute-list>
<URI>
```

主要属性

- BANDWIDTH（必需）, 单位（b/s, bits per second）
- AVERAGE-BANDWIDTH
- CODECS，该值是带引号的字符串，其中包含以逗号分隔的列表格式，指定的媒体类型
- RESOLUTION，分辨率
- FRAME-RATE
- AUDIO，带引号的字符串，其值对应的 TYPE 为 AUDIO 的 #EXT-X-MEDIA 标签的 groupId 属性
- VIDEO，带引号的字符串
- SUBTITLES，带引号的字符串
- CLOSED-CAPTIONS，带引号的字符串

例子

**Master Playlist with Alternative Audio**

```
#EXTM3U
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="aac",NAME="English", DEFAULT=YES,AUTOSELECT=YES,LANGUAGE="en",URI="main/english-audio.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="aac",NAME="Deutsch", DEFAULT=NO,AUTOSELECT=YES,LANGUAGE="de", URI="main/german-audio.m3u8"
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="aac",NAME="Commentary", DEFAULT=NO,AUTOSELECT=NO,LANGUAGE="en", URI="commentary/audio-only.m3u8"
#EXT-X-STREAM-INF:BANDWIDTH=1280000,CODECS="...",AUDIO="aac"
low/video-only.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2560000,CODECS="...",AUDIO="aac"
mid/video-only.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=7680000,CODECS="...",AUDIO="aac"
hi/video-only.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=65000,CODECS="mp4a.40.5",AUDIO="aac"
main/english-audio.m3u8
```

4.3.4.2.1 Alternative Renditions

4.3.4.3 EXT-X-I-FRAME-STREAM-INF

用于指定一个 Media Playlist 包含媒体的 I-frames

4.3.4.4 EXT-X-SESSION-DATA

存放一些 session 数据

4.3.4.5 EXT-X-SESSION-KEY

用于解密

4.3.5 Media or Master Playlist Tags

4.3.5.1 EXT-X-INDEPENDENT-SEGMENTS

表示每个 Media Segment 可以独立解码

4.3.5.2 EXT-X-START

标识一个优选的点来播放这个 Playlist

## 第三方库

- [open-m3u8](https://github.com/iheartradio/open-m3u8)
- [m3u8-parser](https://github.com/carlanton/m3u8-parser)

## 参考

- [M3U8格式讲解及实际应用分析](https://www.cnblogs.com/shakin/p/3870439.html)
- [技术干货|HLS 协议详解及优化技术解析](http://support.upyun.com/hc/kb/article/1030975/)
- [python爬取网站m3u8视频，将ts解密成mp4，合并成整体视频](https://blog.csdn.net/a33445621/article/details/80377424)
- [视频内容加密封装技术研究](http://www.capt.cn/?p=4532)

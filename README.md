# 0 前言

本项目fork自[github.com/sonysuqin/BeQuic](https://github.com/sonysuqin/BeQuic)。

原项目将chromium中的quic实现封装成bequic，并将其嵌入到了ffmpeg。近期发现其中的很多代码，无法在最新版的chromium源码中编译，经过若干调整，最终使其顺利编译通过，因此上传此项目。

注：所有修改，均只在Linux平台（Ubuntu 18.04）上编译并测试通过，其它平台没有验证，但涉及到具体平台的修改应该不大。

# 1 背景

QUIC也就是HTTP3，是Google为了解决HTTP2的一些问题提出的基于UDP的传输方案。HTTP2由于队头堵塞、握手延迟等缺陷并没有普及，相反由于QUIC的优势，Youtube已经全面使用了QUIC。现在去看Youtube的视频，对一个HTTP1/2的请求，Youtube返回的HTTP1/2响应中会携带一个支持QUIC协议的头，Chrome缓存该头中携带的地址，下次对该地址的请求会以QUIC协议发出，之后跟该服务的数据交互将以基于UDP的QUIC协议进行。

QUIC的主要优点是减少握手延迟(号称0RTT)、出色的流控(BBR、CUBIC)、多路复用等，已经成为事实上的下一代HTTP标准。但是目前该标准在实现上有两个版本，一个是IETF的版本，一个是Google自己的版本，这两者目前并不互通。

本文介绍了基于Google的QUIC协议封装的bequic库，并集成到FFmpeg中，让FFmpeg可以通过QUIC协议播放视频。

# 2 代码地址

bequic：

[https://github.com/bookxiao/BeQuic](https://github.com/bookxiao/BeQuic/tree/chromium-6d8828f6)

FFmpeg：
[https://github.com/sonysuqin/FFmpeg](https://github.com/sonysuqin/FFmpeg)

# 3 方案

bequic库封装了Google Chromium Quiche库，对外提供C的接口，因此需要完整下载chromium项目代码用于编译。对FFmpeg，需要增加一个quic协议，主要用于调用bequic库。

## 3.1 bequic  -  Google Quiche封装

增加一个Chromium的动态库工程，将`patch/BUILD.gn`中的内容添加到 `chromium/src/net/BUILD.gn` 相应位置:

```
if (!is_ios) {
  # skip something...
  executable("quic_server") {
    # skip...
  }
+ shared_library("libbequic") {
+   sources = [
+     "tools/quic/basic_streambuf.hpp",
+     "tools/quic/basic_streambuf_fwd.hpp",
+     "tools/quic/be_quic_define.h",
+     "tools/quic/be_quic.h",
+     "tools/quic/be_quic.cc",
+     "tools/quic/be_quic_block.h",
+     "tools/quic/be_quic_block.cc",
+     "tools/quic/be_quic_client.h",
+     "tools/quic/be_quic_client.cc",
+     "tools/quic/be_quic_client_manager.h",
+     "tools/quic/be_quic_client_manager.cc",
+     "tools/quic/be_quic_client_message_loop_network_helper.h",
+     "tools/quic/be_quic_client_message_loop_network_helper.cc",
+     "tools/quic/be_quic_fake_proof_verifier.h",
+     "tools/quic/be_quic_fake_proof_verifier.cc",
+     "tools/quic/be_quic_spdy_client.h",
+     "tools/quic/be_quic_spdy_client.cc",
+     "tools/quic/be_quic_spdy_client_session.h",
+     "tools/quic/be_quic_spdy_client_session.cc",
+     "tools/quic/be_quic_spdy_client_stream.h",
+     "tools/quic/be_quic_spdy_client_stream.cc",
+     "tools/quic/buffer.hpp",
+     "tools/quic/streambuf.hpp",
+   ]
+   deps = [
+     ":net",
+     ":simple_quic_tools",
+     "//base",
+     "//build/win:default_exe_manifest",
+     "//url",
+   ]
+   defines = [ "BE_QUIC_EXPORTS", "BE_QUIC_SHARED_LIBRARY" ]
+ }
  # skip something...
}
```

添加的文件就是bequic库的实现文件(在[BeQuic代码](https://github.com/sonysuqin/BeQuic)的src/chromium目录下)，主要实现了以下接口：
* `be_quic_open`：建立一个QUIC会话，并打开一个流；
* `be_quic_close`：关闭QUIC会话，关闭所有流；
* `be_quic_read`：读QUIC会话当前流的数据；
* `be_quic_seek`：Seek到文件指定位置，可能会打开一个新的流。

> 对seek来说，传统HTTP使用的是带Range头的请求，而Youtube目前并没有使用这个头，而是在URL中携带range参数：http://xx.com/xx.html?……&range=1024-2048&……

> 对BoringSSL与OpenSSL可能的符号冲突，参考这个链接：[https://boringssl.googlesource.com/boringssl/+/HEAD/INCORPORATING.md](https://boringssl.googlesource.com/boringssl/+/HEAD/INCORPORATING.md)，需要编译成动态库并隐藏符号。

## 3.2 FFmpeg  -  增加quic协议

这里在FFmpeg4.1分支上创建了一个4.1.quic分支，可以在[https://github.com/sonysuqin/FFmpeg](https://github.com/sonysuqin/FFmpeg)上查看基于该分支的修改，主要是修改了configure、Makefile，并在libavformat下增加了bequic.c，用于调用bequic库。

>在Windows下，FFMpeg使用MSYS2+MINGW32+GCC编译，chromium使用clang-cl编译，两者的符号不一致，需要使用dlltool等工具对chromium项目编译出的bequic库进行处理，得到GCC可以链接的库。

# 4 编译
## 4.1 Windows
### 4.1.1 编译环境

| 软件| 版本|
|:--|:--|
| Windows | 10 |
| MSYS2 + MINGW|  最新版，还要安装GCC等工具 |
| Chromium |  最新版，版本号6d8828f6a6eea769a05fa1c0b7acf10aca631d4a |
| FFMpeg |  4.1 |

### 4.1.2 目录结构
确保目录结构如下：

```
quic
|-- BeQuic
|-- chromium
`-- FFmpeg
```
### 4.1.3 编译bequic
#### 4.1.3.1 下载bequic源码

在quic目录下，执行：
```
git clone https://github.com/bookxiao/BeQuic.git
```

#### 4.1.3.2 下载chromium源码

在quic目录下，按照chromium的[官方编译文档](https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md)，下载chromium代码(需要一个比较好的VPN)。

#### 4.1.3.3 准备代码

使用MSYS2进入BeQuic/patch目录下，执行

- 把BeQuic/patch/BUILD.gn内容插入到chromium/src/net/BUILD.gn合适位置；
- 把BeQuic/src/chromium下的文件拷贝到chromium/src/net/tools/quic目录下；

#### 4.1.3.4 生成工程

在chromium/src下执行：

```
gn gen out/Debug--args="is_debug=true is_component_build=false target_cpu=\"x86\""
```

这里可以决定是产生Debug版还是Release版。
#### 4.1.3.5 编译 bequic库
在chromium/src下执行：
```
ninja -C out\Debug libbequic
```
#### 4.1.3.6 编译quic_server  - 用于测试
在chromium/src下执行：
```
ninja -C out\Debug quic_server
```
### 4.1.4 编译FFmpeg
#### 4.1.4.1 安装依赖
安装GCC、SDL2等FFmpeg通常需要依赖的工具，主要参考了以下这些网页：
[《msys2和SDL2环境搭建》](https://dongqiceo.github.io/the-post-9982/)
[《windows下编译FFMPEG篇》](https://blog.csdn.net/listener51/article/details/81605472)
[《Windows10平台编译ffmpeg 4.0.2，生成ffplay》](https://www.cnblogs.com/harlanc/p/9569960.html)
#### 4.1.4.2 下载支持bequic的FFmpeg源码
在quic目录下，执行

```
git clone https://github.com/sonysuqin/FFmpeg.git
```
#### 4.1.4.3 处理bequic库的符号
将BeQuic/script/gen_a.sh拷贝到chromium/src/out/Debug目录下，执行：

```
./gen_a.sh
```

该脚本很简单：

```
gendef libbequic.dll
dlltool --kill-at -d libbequic.def --dllname libbequic.dll -l libbequic.a
```
使用gendef产生libbequic.dll的def文件，然后用dlltool产生GCC可以链接的libbequic.a。

#### 4.1.4.4 configure
在FFmpeg目录下，执行：
```
mkdir build
cd build
../configure --disable-static --enable-shared --enable-gpl --enable-version3 --enable-sdl --enable-debug=3 --disable-optimizations --disable-mmx --disable-stripping --arch=x86 --enable-libbequic --extra-cflags=-I/d/work/google/chromium/src/net/tools/quic --extra-ldflags=-L/d/work/google/chromium/src/out/Debug --extra-libs=-lbequic
```
注意修改--extra-cflags、--extra-ldflags为实际的bequic库的头文件、库文件的路径。

#### 4.1.4.5 编译
在FFmpeg/build目录下，执行：

```
make && make install
```
注意把chromium/src/out/Debug/libbequic.dll拷贝到ffplay运行目录。
## 4.2 Android
### 4.2.1 编译环境
| 软件| 版本|
|:--|:--|
| Ubuntu | 16.04 |
| NDK |  r17c |
| Chromium |  最新版，版本号e57cf4b40708a719439ad3895279b7de1feb62a8|
| FFMpeg |  4.1.quic |
### 4.2.2 目录结构
确保目录结构如下：
```
quic
|-- BeQuic
|-- chromium
`-- FFmpeg
```
### 4.2.3 编译bequic
#### 4.2.3.1 下载bequic源码
在quic目录下，执行：
```
git clone https://github.com/sonysuqin/BeQuic.git
```
#### 4.2.3.2 下载chromium源码
在quic目录下，按照chromium的[官方编译文档](https://chromium.googlesource.com/chromium/src/+/master/docs/android_build_instructions.md)，下载chromium代码(需要一个比较好的VPN)。
#### 4.2.3.3 打bequic补丁
进入BeQuic/patch目录下，执行
```
mv BUILD.gn BUILD.gn.bak
mv BUILD.gn.Android BUILD.gn
./patch.sh
mv BUILD.gn BUILD.gn.Android
mv BUILD.gn.bak BUILD.gn
```
目的：
* 将BeQuic/patch/BUILD.gn.Android覆盖chromium/src/net目录下的BUILD.gn文件；
* 把BeQuic/src/chromium下的文件拷贝到chromium/src/net/tools/quic目录下；

> 补丁并不修改chromium的源代码。

#### 4.2.3.4 生成工程
在chromium/src下执行：
```
gn gen out/Release --args="is_debug=false is_component_build=false target_os=\"android\"  target_cpu=\"arm\""
```
这里可以决定是产生Debug版还是Release版。
#### 4.2.3.5 编译 bequic库
在chromium/src下执行：

```
ninja -C out/Release libbequic
```
### 4.2.4 编译FFmpeg
#### 4.2.4.1 下载支持bequic的FFmpeg源码
在quic目录下，执行

```
git clone https://github.com/sonysuqin/FFmpeg.git
```

#### 4.2.4.2 编译
在FFmpeg/build目录下，执行：

```
./build_android.sh
```
在编译脚本中，需要指定NDK工具链的路径，libbequic库的头文件、库路径等(如果按照上述目录结构就不用修改)，脚本会将编译产生的所有输出放在当前目录的android目录下。
#### 4.2.4.3 Androd端简单测试
用Android Studio(3.4)打开BeQuic/test/android目录下的TestFFmpegQuic工程，将4.2.4.2节编译产生的所有.so库拷贝到BeQuic/test/android/TestFFmpegQuic/FFmpegQuic/src/main/jni/FFmpeg/lib/armeabi-v7a目录下，连接Android手机，编译、运行。


## 4.3 Linux
### 4.3.1 编译环境
| 软件| 版本|
|:--|:--|
| Ubuntu| 18.04 |
| gcc/g++ |  Ubuntu 8.4.0-1ubuntu1~18.04) |
| Chromium |  最新版，版本号6d8828f6a6eea769a05fa1c0b7acf10aca631d4a|
| FFMpeg |  4.1.quic |
### 4.3.2 目录结构
确保目录结构如下
```
quic
|-- BeQuic
|-- chromium
`-- FFmpeg
```
### 4.3.3 编译bequic
#### 4.3.3.1 下载bequic源码
创建并进入quic目录，执行
```
git clone https://github.com/bookxiao/BeQuic.git
```
#### 4.3.3.2 下载chromium源码

在quic目录下，按照chromium的官方[编译文档](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md)，下载chromium代码(需要一个比较好的VPN)。

#### 4.3.3.3 准备代码

- 将BeQuic/patch/BUILD.gn内容插入到chromium/src/net目录下的BUILD.gn文件相应位置；
- 把BeQuic/src/chromium下的文件拷贝到chromium/src/net/tools/quic目录下；

#### 4.3.3.4 生成工程
在chromium/src下执行：
```
gn gen out/linux_release_x64_notcmalloc --args="is_debug=false is_component_build=false target_os=\"linux\" target_cpu=\"x64\" use_allocator=\"none\""
```
这里可以决定是产生Debug版还是Release版；
user_allocator="none" ：表示编译时malloc等系统库使用系统库函数，而不调用chromium自己实现的tcmalloc。

#### 4.3.3.5 编译bequic库

在chromium/src下执行：

```
ninja -C out/linux_release_x64_notcmalloc libbequic
```

### 4.3.4 编译FFmpeg
#### 4.3.4.1 下载支持bequic的FFmpeg源码
在quic目录下，执行

```
git clone https://github.com/sonysuqin/FFmpeg.git
```
切换ffmepg版本为： **4.1.quic** 

#### 4.3.4.2 configure

在FFmpeg下创建build目录，并进入FFmpeg/build目录，执行

```
../configure --disable-static --enable-shared --enable-gpl --enable-version3 --enable-sdl --enable-debug=3 --disable-optimizations --disable-mmx --disable-stripping --arch=x64 --enable-libbequic --extra-cflags="-I../../quic/chromium/src/net/tools/quic -fPIC" --extra-ldflags="-L../../quic/chromium/src/out/linux_release_x64 -fPIC" --extra-libs=-lbequic --prefix=../linux
```
注意修改–extra-cflags、–extra-ldflags为实际的bequic库的头文件、库文件的路径.。

如果出现`C compiler test failed.`错误，可以将`configure`中的这一行修改成：

```sh
diff --git a/configure b/configure
index a1cff7a4ce..dd7742951a 100755
--- a/configure
+++ b/configure
@@ -1101,7 +1101,8 @@ test_ld(){
     test_$type $($cflags_filter $flags) || return
     flags=$($ldflags_filter $flags)
     libs=$($ldflags_filter $libs)
-    test_cmd $ld $LDFLAGS $LDEXEFLAGS $flags $(ld_o $TMPE) $TMPO $libs $extralibs
+    #test_cmd $ld $LDFLAGS $LDEXEFLAGS $flags $(ld_o $TMPE) $TMPO $libs $extralibs
+    test_cmd $ld $LDFLAGS $LDEXEFLAGS $flags $(ld_o $TMPE) $TMPO $libs
 }
```

#### 4.3.4.3 编译

在FFmpeg/build目录下，执行

```
make && make install
```

#### 4.3.4.4 测试
将libbequic.so拷贝到FFmpeg/build/lib下，执行

```
export LD_LIBRARY_PATH=<filepath>/ffmpeg/build/lib
```
然后在FFmpeg/build执行

```
./ffplay quic://www.example.org:6121 -timeout 1000 -verify_certificate 1
```

## 4.4 iOS
### 4.4.1 编译环境
| 软件| 版本|
|:--|:--|
| mac| 10.14 |
| clang|  x86_64-apple-darwin18.0.0|
| Chromium |  最新版，版本号eca01417caf1e26ce41c88f2804291ddee75f5ed|
| xcode | 10.1 |
### 4.4.2 目录结构
确保目录结构如下
```
quic
|-- BeQuic
|-- chromium
```
### 4.4.3 编译bequic
#### 4.4.3.1 下载bequic源码
在quic目录下，执行：
```
git clone https://github.com/sonysuqin/BeQuic.git
```
#### 4.4.3.2 下载chromium源码
在quic目录下，按照chromium的官方[编译文档](https://chromium.googlesource.com/chromium/src/+/master/docs/ios/build_instructions.md)，下载chromium代码(需要一个比较好的VPN)。
#### 4.4.3.3 打bequic补丁
进入BeQuic/patch目录下，执行

```
mv BUILD.gn BUILD.gn.bak
mv BUILD.gn.ios BUILD.gn
./patch.sh
mv BUILD.gn BUILD.gn.ios
mv BUILD.gn.bak BUILD.gn
```
目的：
- 将BeQuic/patch/BUILD.gn.ios覆盖chromium/src/net目录下的BUILD.gn文件；
- 把BeQuic/src/chromium下的文件拷贝到chromium/src/net/tools/quic目录下；

>补丁并不修改chromium的源代码。
#### 4.4.3.4 生成工程
在chromium/src下执行：
```
gn gen out/ios_arm64_shared --args="is_debug=false is_component_build=false target_os=\"ios\" target_cpu=\"arm64\" ios_code_signing_identity=\"DB1A2124EA21DA7F2BECD69186C8DCFB4A1B7CC7\"" --ide=xcode
```
ios_code_signing_identity代表签名，使用下面命令查看本机所有签名
```
find-identity -v -p codesigning
```
#### 4.4.3.5 编译bequic库
在chromium/src下执行

```
ninja -C out/ios_arm64_shared libbequic
```
注意：如果使用编译后的库，程序可以正常调用，但是执行后会出现下面的错误：
*dyld: (Library not loaded: ./libbequic.dylib ... Reason: image not found)*
在终端下查看，执行
```
$otool -L libbequic.dylib 
		libbequic.dylib:
			./libbequic.dylib (compatibility version 0.0.0, current version 0.0.0)
			/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation (compatibility version 150.0.0, current version 1560.10.0)
			/System/Library/Frameworks/CoreGraphics.framework/CoreGraphics (compatibility version 64.0.0, current version 1245.9.2)
			/System/Library/Frameworks/CoreText.framework/CoreText (compatibility version 1.0.0, current version 1.0.0)
			/System/Library/Frameworks/Foundation.framework/Foundation (compatibility version 300.0.0, current version 1560.10.0)
			/System/Library/Frameworks/UIKit.framework/UIKit (compatibility version 1.0.0, current version 61000.0.0)
			/System/Library/Frameworks/CFNetwork.framework/CFNetwork (compatibility version 1.0.0, current version 975.0.3)
			/System/Library/Frameworks/MobileCoreServices.framework/MobileCoreServices (compatibility version 1.0.0, current version 935.2.0)
			/System/Library/Frameworks/Security.framework/Security (compatibility version 1.0.0, current version 58286.222.2)
			/System/Library/Frameworks/SystemConfiguration.framework/SystemConfiguration (compatibility version 1.0.0, current version 963.200.27)
			/usr/lib/libresolv.9.dylib (compatibility version 1.0.0, current version 1.0.0)
			/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.200.5)
			/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
```
> ios动态库的链接路径不能使用相对路径，否则无法找到链接文件。

修改动态库的查找路径
```
install_name_tool -id @rpath/libbequic.dylib libbequic.dylib
```
再用命令otool -L libbequic.dylib查看，发现已经修改。
### 4.4.4 编译测试程序
使用xcode打开Bequic/test/ios/quic_shared_test/quic_test.xcodeproj，编译后测试。

## 4.5 TBD
Mac OSX.

# 5 测试

首先，我们需要有一个支持quic播放的流媒体服务器和一条流。

> 可以使用[nginx-quic](https://github.com/evansun922/nginx-quic/blob/master/README-CN.md) 或者[腾讯云quic测试小程序](https://github.com/tencentyun/qcloud-documents/blob/master/product/%E8%A7%86%E9%A2%91%E6%9C%8D%E5%8A%A1/%E7%9B%B4%E6%92%AD/%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5/QUIC%E7%9B%B4%E6%92%AD.md#push)来产生支持quic的流

假定我们有条流地址为：`https://example.com/live/teststream.flv`

然后我们启动`ffplay`:

```sh
$ ffplay quics://example.com/live/teststream.flv
```

注意：如果直接传入http或https开头的留地址给ffplay，是不会触发quic模块的。需要将 http 换成 quic, https 换成 quics，才能触发quic模块。

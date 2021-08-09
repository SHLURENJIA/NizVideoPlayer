# IjkPlayer 初步使用（编译自定义so）

## IjkPlayer 初步使用
[IjkPlayer](https://github.com/bilibili/ijkplayer),其实已经很久没有发布release了，但是这个毕竟是国内很出名且占用率较高的库，所以还是采用了他。而且他是基于ffmpeg，软解效果很好，基本支持所有的协议。
IjkPlayer在jar层一般可以直接使用，但是so层，往往需要自己编译修改，得到自己要的so来集成使用。**比如ijkplayer默认就是不支持https！**

## 编译前准备
1. 安装编译环境，安装yasm，git。（Yasm是一个完全重写的NASM汇编，简单理解就是一个汇编指令集
2. 下载ijkplayer仓库源码
3. 配置ndk，需要配置ANDROID_NDK环境变量，否则后面编译会有问题。ndk版本我使用了android-ndk-r10e
```shell script
# install homebrew, git, yasm
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install git
brew install yasm

# add these lines to your ~/.bash_profile or ~/.profile
# export ANDROID_SDK=<your sdk path>
# export ANDROID_NDK=<your ndk path>

# on Cygwin (unmaintained)
# install git, make, yasm

```

## 编译so
1. 选择编译的配置
> ln 表示建立给两个文件建立链接， -s 表示软链接
```shell script
cd config
rm module.sh
# 更多的编解码器/格式
ln -s module-default.sh module.sh
# 较少的编解码器/格式（但是包括hevc
ln -s module-lite-hevc.sh module.sh
# 较少的编解码器/格式（默认
ln -s module-lite.sh module.sh
```

2. 编译openSSL和FFMPEG
回到项目根目录，执行项目根目录下的初始化脚本，脚本以init-<platform>-xxx.sh形式存在。执行时git会clone下载对应的代码，耗时可能较久。建议科学上网，可缩减时间。
openssl是https支持的库，需要把他一并集成进去，这样ijkplayer才会支持https。
```shell script
./init-android-openssl.sh
./init-android.sh
```

然后cd 进入 android/contrib 目录，执行清除脚本，编译需要的库。以下是编译openssl和ffmpeg的库，openssl用以支持https，ffmpeg用来支持支持视频编解码。
> all 参数表示全部架构的 so，可以将 all 替换成 arm64 等（armv5|armv7a|arm64|x86|x86_64）。我这里出于时间考虑直接编译了arm64。
```shell script
./compile-openssl.sh clean//清除
./compile-ffmpeg.sh clean//清除
./compile-openssl.sh all//编译
./compile-ffmpeg.sh all//编译
```

运行完了就有Finished提示，说明编译成功了。编译成功后会在build的对应架构目录libs下生成.a动态库文件

PS-1. 报错 `Unknown option "--disable-ffserver" `
在[issues](https://github.com/bilibili/ijkplayer/issues/4690)中有解决方案：将config/module.sh中的两句注释就可以了
```shell
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-vda"
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-ffserver"
```


PS-2. 报错`hevc_mvs.c:207:15: error: 'x0000000' undeclared (first use in this function)``
也是在[issues](https://github.com/bilibili/ijkplayer/issues/4093)找到解决方案：加上下面那句到config/module.sh中即可。
```shell
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --disable-linux-perf"
```


3. 编译生成ijkplayer的so
切换到android目录下，执行脚本生成ijkplayer相关的so，执行成功后会在android/ijkplayer的对应目录下生成ijkplayer的so。
如：android/ijkplayer/ijkplayer-arm64/src/main/libs/arm64-v8a。
```shell script
./compile-ijk.sh all
```

以上就成功编译了ijkplayer，还是带https的。接下来我们要集成验证一下。
复制架构的文件，到Android项目的main/jniLibs目录下，也可以直接使用ijkplayer的demo，然后找个https的视频播放一下。
我这边用的是ijk的sample的demo，在 `SampleMediaListFragment` 的adapter上增加了个https的视频测试。

如果不支持的话，会报错
`IJKMEDIA: https protocol not found, recompile FFmpeg with openssl, gnutls or securetransport enabled.`

参考资料：
1.[Ijkplayer编译自定义so](https://github.com/CarGuo/GSYVideoPlayer/blob/master/doc/BUILD_SO.md)
2.[Ijkplayer官方README](https://github.com/bilibili/ijkplayer/blob/master/README.md#before-build)
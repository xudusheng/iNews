###  目录
>* 摘要
>* 播放：ijkplayer、VLC
>* 服务器：nginx+rtmp+ffmpeg
>* 推流：LFLiveKit、ffmpeg
>* APP直播模拟
>* 参考资料


### 1、摘要
&emsp;&emsp;最近又重新翻看了一下iOS相关的点播与直播资料，也踩了不少坑。网上也有不少相关资料，但是完整直播流程一直走不通，要么是电脑推流手机播放，要么是电脑推流电脑播放，至于手机推流的完整demo相对较少，无法很直观的体会完整的手机直播，即手机推流与手机播放。    
&emsp;&emsp;本例将借助nginx+rtmp+ffmpeg搭建一个简单的直播系统，通过手机采集音视频，经过简单的图像处理和编码，再将流推到自己搭建的服务器上（顺带介绍一下电脑推流），最后通过手机和电脑进行播放了。  

&emsp;&emsp;一个完整的直播系统需要涉及到的技术及流程主要包括以下方面：

    采集 => 图像处理 => 编码 => 推流 => CDN分发 => 拉流 => 解码 => 播放 => 聊天互动。    

&emsp;&emsp;在本例中，采集=>滤镜处理=>编码=>推流由LFLiveKit来完成，其中图像处理交给GPUImage库完成，而LFLiveKit已经集成了GPUImage库；CDN分发就是搭建的本地服务器；拉流=>解码=>播放由ijkplayer库来完成；聊天互动属于IM范畴，这里就讨论了，有兴趣的朋友可以自行搜索。这里重点是操作，没有太多涉及理论的东西，目的是希望通过一个简单的例子，加深对直播的理解。后续也会慢慢补上直播中各个技术的理论知识与demo。

### 2、播放环境搭建：ijkplayer、VLC
&emsp;&emsp; VLC：电脑版的播放器，用于模拟在电脑端播放。   
&emsp;&emsp; ijkplayer：是基于FFmpeg的跨平台播放器框架，github地址：https://github.com/Bilibili/ijkplayer， iOS版的播放器将使用ijkplayer框架进行集成。  

&emsp;&emsp; 先提供一个播放源数据：http://116.211.167.106/api/live/aggregation?uid=133825214&interest=1, 复制链接到浏览器中打开，会返回一个json格式的数据，其中一个stream_addr的值就是一个播放源。（感谢@袁峥Seemygo提供）。
![image](http://ohlldt20k.bkt.clouddn.com/hls_1_1.png)


#### &emsp;&emsp;2.1、Mac端使用VLC进行播放 
&emsp;&emsp;百度下载mac版的VLC进行安装，打开VLC，File -> Open Network…
![image](http://ohlldt20k.bkt.clouddn.com/hls_1_7.png)

#### &emsp;&emsp;2.2、iOS集成ijkplayer
#### &emsp;&emsp;2.2.1、ijkplayer集成

##### &emsp;&emsp; a> 下载ijkplayer源码：(下载地址:https://github.com/Bilibili/ijkplayer)  
##### &emsp;&emsp; b> 导入ffmpeg
&emsp;&emsp;ijkplayer是基于ffmpeg这个库的，因此需要导入ffmpeg库  
&emsp;&emsp;打开终端，cd到ijkplayer所在目录，可以看到init-ios.sh的脚本文件，运行脚本文件:

    $ ./init-ios.sh

&emsp;&emsp;等待一段时间....运行完成以后，ios文件夹下面会多出来四个文件夹，分别是ffmpeg-arm64、ffmpeg-armv7、ffmpeg-i386、ffmpeg-x86_64。  
##### &emsp;&emsp; c> 编译ffmpeg库  

    //依次执行
    //进入ios文件夹
    $ cd ios
    
    //删除一些文件和文件夹，为编译ffmpeg.sh做准备，在编译ffmpeg.sh的时候，会自动创建刚刚删除的那些文件，为避免文件名冲突，因此在编译ffmpeg.sh之前先删除等会会自动创建的文件夹或者文件
    $ ./compile-ffmpeg.sh clean
    
    //真正的编译各个平台的ffmpeg库，并生成所有平台的通用库。
    $ ./compile-ffmpeg.sh all


&emsp;&emsp; 双击进入ios文件夹，打开IJKMediaPlayer工程，查看Class-IJKFFMoviePlayerController-ffmpeg-lib下的.a文件，如果文件一片红，说明ffmpeg的编译失败，请重复b、c操作。  
![image](http://ohlldt20k.bkt.clouddn.com/hls_1_8.png)

##### &emsp;&emsp; d> 编译IJKMediaFramework  
&emsp;&emsp;集成ijkplayer的方法有两种，一种是将以上运行成功的IJKMediaPlayer工程中的IJKMediaPlayer.xcodeproj直接导入目标工程，在这里不做介绍;  
&emsp;&emsp;另外一种就是将ijkplayer打包成framework导入工程。  

&emsp;&emsp; e.1 打开IJKMediaPlayer工程  

&emsp;&emsp; e.2 设置工程的scheme
    选择product-Scheme-Edit Scheme
    选择run下面的Build Configuration为release，如图
    ![image](http://ohlldt20k.bkt.clouddn.com/hls_1_2.png)
    
&emsp;&emsp; e.3 设置好scheme以后，分别选择真机和模拟器进行编译（重要：编译前记得clean一下），编译完成以后，选择Product下面的IJKMediaFramework.framework，进入finder，如图
![image](http://ohlldt20k.bkt.clouddn.com/hls_1_6.png)

Release-iphoneos是真机版本的framework，只能跑真机，不能跑模拟器；  
Release-iphonesimulator是模拟器版本的framework，只能跑模拟器，不能跑真机；
如果希望真机与模拟器都能运行，那么就需要对这两个framework进行合并。  

&emsp;&emsp; e.4 合并真机与模拟器版本的framework  
    
    //注意：合并的目标是
    Release-iphoneos/IJKMediaFramework.framework/IJKMediaFramework  
    Release-iphonesimulator/IJKMediaFramework.framework/IJKMediaFramework
    
    //合并代码
    //$ lipo -create "真机版路径" "模拟器版路径" -output "合并后的版本路径"
    <!--比如我的：-->
    $ lipo -create /Users/Hmily/Desktop/framework/Release-iphoneos/IJKMediaFramework.framework/IJKMediaFramework /Users/Hmily/Desktop/framework/Release-iphonesimulator/IJKMediaFramework.framework/IJKMediaFramework -output /Users/Hmily/Desktop/framework/IJKMediaFramework
    
    复制Release-iphoneos文件夹，粘贴并命名为Release-iphonesimulator-OS
    将Release-iphonesimulator-OS/IJKMediaFramework.framework/IJKMediaFramework
    替换为刚刚合并生成的IJKMediaFramework
    至此，真机与模拟器版的framework制作完成。
    
    ![image](http://ohlldt20k.bkt.clouddn.com/hls_1_3.png)
    
&emsp;&emsp; e.5 将IJKMediaFramework.framework集成到xcode工程中
    将IJKMediaFramework.framework添加到自己的工程中，并添加以下库支持
    
    AudioToolbox.framework         
    AVFoundation.framework
    CoreMedia.framework
    CoreVideo.framework
    libbz2.tbd
    libz.tbd
    MediaPlayer.framework
    MobileCoreServices.framework
    OpenGLES.framework
    VideoToolbox.framework


#### &emsp;&emsp;2.3、编写iOS代码
    - (void)viewDidLoad {
        [super viewDidLoad];
        self.view.backgroundColor = [UIColor whiteColor];
        NSString * videoUrl = self.videoInfo[@"stream_addr"];
        //videoUrl = @"rtmp://live.hkstv.hk.lxdns.com:1935/live/stream1555";
        self.player = [[IJKFFMoviePlayerController alloc] initWithContentURLString:videoUrl
                                                                   withOptions:nil];
        [_player prepareToPlay];

        [self.view addSubview:_player.view];
    }
    - (void)viewWillDisappear:(BOOL)animated{
       [super viewWillDisappear:animated];
       [_player pause];
       [_player stop];
    }

    - (void)dealloc{
       _player = nil;
    }



### 3、服务器搭建：nginx+rtmp+ffmpeg
上面我们完成了播放器的搭建，接下来我们搭建一个属于自己的服务器。
3.1、安装Homebrew
打开终端，输入： 

     $ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

3.2、安装nginx

依次输入以下的命令：

    //从github下载Nginx到本地,增加home-brew对nginx的扩展：   
    $ brew tap homebrew/nginx

    //安装Nginx服务器和rtmp模块:
    $ brew install nginx-full --with-rtmp-module
    
    //启动nginx服务器
    $ nginx
    
    在浏览器地址栏输入：http://localhost:8080 （直接点击）
如果出现下图, 则表示安装成功。  
![image](http://ohlldt20k.bkt.clouddn.com/hls_1_10.png)

3.3、配置rtmp

    //查看nginx信息
    $ brew info nginx-full
    
![image](http://ohlldt20k.bkt.clouddn.com/hls_1_9.png)


打开/usr/local/etc/nginx/nginx.conf文件，在最好面插入以下代码：
    rtmp {
        server {
            listen 1990;            #监听接口号
            
            #RTMP协议
            application liveApp {   #app名称
               live on;
               record off;          #不记录数据
            }
            
            #HLS协议
            application hls{
               live on;             #开启实时
               hls on;              #开始HLS协议
               hls_fragment 1s;     #切片时长
               
               hls_path /usr/local/var/www/hls;     #文件保存路劲
               #record off;          #不记录数据      
            }
        }
    }

重新加载nginx的配置文件：

    $ nginx -s reload
    
    至此服务器的配置就算完成了，接下来我们进行推流，并进行直播测试
    
    
### 4、推流测试
4.1、使用ffmepg推流测试  
安装ffmpeg
输入：

    $ brew install ffmpeg

使用ffmepg对视频切片  
准备一个视频文件，终端cd到该文件所在的目录：

    $ cd /Users/Hmily/Desktop/hls
    $ ffmpeg -i mtv.mp4 -c:v libx264 -c:a copy -f hls mtv.m3u8

接着是漫长的等待，ffmpeg正在将mtv.mp4切成一个个很小的ts文件，并生成一个mtv.m3u8的索引文件。
切片完成以后，将hls文件夹拷贝到/usr/local/var/www目录下。  
打开浏览器，输入http://localhost:8080/hls/mtv.m3u8，回车。



4.2、iOS直播APP推流测试


### 5、APP直播模拟

### 参考资料

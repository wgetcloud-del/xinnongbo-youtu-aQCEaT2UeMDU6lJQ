
随着国际政治经济形势的变化，尤其是中美科技竞争日益激烈，软件信创国产化已经迫在眉睫。在这种大环境下，我们将现有的Windows版软件逐步迁移到信创国产化基础设施上，适配国产操作系统（如银河麒麟、统信UOS）、国信芯片（如飞腾、鲲鹏、海光、龙芯、麒麟）以及国产DB。


我们经常有这样的需求，比如需要在银河麒麟或统信UOS上实现RTMP推流摄像头视频和麦克风声音到流媒体服务器（如nginx或srs），那么这个要如何实现了？


## 一. 技术方案


要完成这个功能，具体来说，需要解决如下几个技术问题：


（1）麦克风数据采集。


（2）摄像头数据采集。


（3）音频数据AAC编码。


（4）视频数据H264编码。


（5）将编码后的数据按RTMP协议推送给流媒体服务器。


（6）通过时间戳（PTS）保证音频视频的同步。


我们使用跨平台的 .NET Core （C\#），跨平台的UI框架Avalonia，再借助 LinuxCapture 和 NPusher.NetCore 这两个组件，就很容易采集到麦克风和摄像头的数据，并且将它们推流到流媒体服务器上。


我们先看看推流程序在银河麒麟上的运行效果：


 ![](https://img2024.cnblogs.com/blog/513369/202410/513369-20241023091816856-390327372.png)


 两个下拉列表可以选择要使用的麦克风和摄像头设备。


点击“开始”按钮，麦克风和摄像头将开始采集数据，并推流至流媒体Server。


如果中途网络断开，推流将会中断，并尝试自动重连，重连成功后，将恢复推流。


点击“结束”按钮，则将结束音视频采集和推流。


## 二.具体实现


（1）ICameraCapturer是摄像头视频采集组件；IMicrophoneCapturer是麦克风声音采集组件。


（2）我们可以通过调用CapturerFactory的CreateXXXX方法来创建对应的采集器实例。


（3）得到采集器实例后，调用Start方法，即可开始采集；调用Stop方法，即停止采集。


（4）采集得到的数据，将通过相应的事件（ImageCaptured、AudioCaptured）暴露出来，我们预定这些事件，即可拿到采集的数据。


（5）将拿到的数据喂给IStreamPusher，就会将其推流到指定的流媒体服务器。


我们这里列一下核心代码，完整的代码大家可以从文末下载源码进行了解。


创建并启动采集器：




```
            //摄像头采集器
            this.cameraCapturer = CapturerFactory.CreateCameraCapturer(cameraIndex, videoSize, frameRate);
            this.cameraCapturer.ImageCaptured += CameraCapturer_ImageCaptured;
            this.cameraCapturer.CaptureError += CameraCapturer_CaptureError;
            //麦克风采集器
            this.microphoneCapturer = CapturerFactory.CreateMicrophoneCapturer(micIndex);
            this.microphoneCapturer.AudioCaptured += MicrophoneCapturer_AudioCaptured;
            this.microphoneCapturer.CaptureError += MicrophoneCapturer_CaptureError;
 
            this.microphoneCapturer.Start();
            this.cameraCapturer.Start();
```


创建并启动推流器： 




```
           string nginxServerIP = ConfigurationManager.AppSettings["NginxServerIP"]; 
           int nginxServerPort = int.Parse(ConfigurationManager.AppSettings["NginxServerPort"]); 
           string rtmpUrl = $"rtmp://{nginxServerIP}:{nginxServerPort}/hls/{streamID}";
           this.streamPusher.UpsideDown4RGB24 = true;
           this.streamPusher.Initialize(rtmpUrl, videoSize.Width, videoSize.Height, InputAudioDataType.PCM, InputVideoDataType.RGBA, this.channelCount);
```


将采集到的数据喂给推流器：




```
private void CameraCapturer_ImageCaptured(byte[] agba32Data)
{ 
    if (this.isRecording)
    {
        this.streamPusher.PushVideoFrame(agba32Data); 
        UiSafeInvoker.ActionOnUI(() =>
        {
            WriteableBitmap writeableBitmap = CreateBitmapFromPixelData(agba32Data, videoSize.Width, videoSize.Height);
            img.Source = writeableBitmap;
        }); 
    }
}
 
private void MicrophoneCapturer_AudioCaptured(byte[] pcm)
{
    if (this.isRecording)
    {
        this.streamPusher.PushAudioFrame(pcm);
    }
}
```


推流器内部会对音视频数据进行编码，并依据RTMP协议发送给流媒体服务器。


停止推流： 




```
private void FinishRecorded(bool success)
{ 
    this.RecordState_Changed(false);
    this.cameraCapturer?.Stop();
    this.cameraCapturer?.Dispose();
    this.microphoneCapturer?.Stop();
    this.microphoneCapturer?.Dispose();
    this.streamPusher?.Close();
    string tip = success ? "推流停止！" : "推流器断开，推流停止！";
    ShowStateMsg(tip); 
}
```


## 三. 部署运行


如果要在银河麒麟或统信UOS上运行这里的RTMP推流程序，则需要现在目标操作系统上安装.NET Core 3\.1。


然后将VS生成目录下的 netcoreapp3\.1 文件夹拷贝到目标电脑上，进入netcoreapp3\.1文件夹，打开终端，并在终端中输入如下命令：




```
dotnet Oraycn_Avalonias_PusherDemo.Desktop.dll
```


回车运行后，就会出现前面截图的UI界面，然后我们就可以预览摄像头，并开始推流麦克风摄像头了。


## 四. 源码下载


[Oraycn.Avalonias.PusherDemo.rar](https://github.com)


源码中包含的非托管库是X64架构的，如果需要在其它架构的国产芯片上运行该程序，可以联系我获取对应架构的非托管库。


 


 本博客参考[飞数机场](https://ze16.com)。转载请注明出处！

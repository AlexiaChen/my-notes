# RGB， YUV像素数据处理

这些像素数据就是解压H264/265等数据后的数据了。

![[Pasted image 20220703200508.png]]

咱先了解一下YUV420P像素数据中的Y,U,V分量。相当于也是学习YUV420P的文件二进制格式了。

```cpp
/**
 * Split Y, U, V planes in YUV420P file.
 * @param url  Location of Input YUV file.
 * @param w    Width of Input YUV file.
 * @param h    Height of Input YUV file.
 * @param num  Number of frames to process.
 *
 */
int simplest_yuv420_split(char *url, int w, int h,int num){
	FILE *fp=fopen(url,"rb+");
	FILE *fp1=fopen("output_420_y.y","wb+");
	FILE *fp2=fopen("output_420_u.y","wb+");
	FILE *fp3=fopen("output_420_v.y","wb+");
 
	unsigned char *pic=(unsigned char *)malloc(w*h*3/2);
 
	for(int i=0;i<num;i++){
 
		fread(pic,1,w*h*3/2,fp);
		//Y
		fwrite(pic,1,w*h,fp1);
		//U
		fwrite(pic+w*h,1,w*h/4,fp2);
		//V
		fwrite(pic+w*h*5/4,1,w*h/4,fp3);
	}
 
	free(pic);
	fclose(fp);
	fclose(fp1);
	fclose(fp2);
	fclose(fp3);
 
	return 0;
}

```

从代码上可以看到，如果视频帧的宽高分别是w和h。那么一帧YUV420P像素数据就占用$w*h*3/2$ 个字节数据，其中前$w*h$个字节存储Y分量，紧随其后的$w*h/4$个字节存储U分量，
最后的$w*h/4$个字节存储V分量。

调用上面函数的简单实例:

```cpp
simplest_yuv420_split("lena_256x256_yuv420p.yuv",256,256,1);
```

代码运行后，将会把一张分辨率为256x256的名称为lena_256x256_yuv420p.yuv的YUV420P格式的像素数据文件分离成为三个文件。

```txt
output_420_y.y：纯Y数据，分辨率为256x256。

output_420_u.y：纯U数据，分辨率为128x128。

output_420_v.y：纯V数据，分辨率为128x128。
```

播放YUV文件格式的播放器很多，比如 [IENT/YUView: The Free and Open Source Cross Platform YUV Viewer with an advanced analytics toolset (github.com)](https://github.com/IENT/YUView)

上面运行后的代码结果示意图:

![[Pasted image 20220703202251.png]]

YUV的原图

![[Pasted image 20220703202305.png]]
output_420_y.y 的Y分量图
![[Pasted image 20220703202318.png]] 
output_420_u.y的U分量图
![[Pasted image 20220703202331.png]]
output_420_v.y的V分量图

文件后缀都是y，这里的U，V分量在YUV播放器中也是当作Y分量进行播放的。





为什么会有YUV420P, YUV444P等这些东西呢？它们有什么不同？那么请看

[图像原始格式(YUV444 YUV422 YUV420)一探究竟 - mcdull^0^ - 博客园 (cnblogs.com)](https://www.cnblogs.com/tid-think/p/10616789.html)

[YUV图解 （YUV444, YUV422, YUV420, YV12, NV12, NV21） - schips - 博客园 (cnblogs.com)](https://www.cnblogs.com/schips/p/12099550.html)

[[2 of 6] Comparison between H.264 YUV420 and YUV444 – GRID4ALL (sschaber.de)](http://sschaber.de/2018/12/03/2-of-6-comparison-between-h-264-yuv420-and-yuv444/)



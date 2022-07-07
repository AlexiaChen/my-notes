# RGB， YUV像素数据处理

这些像素数据就是解压H264/265等数据后的数据了。

![[Pasted image 20220703200508.png]]


YUV中的Y(Luma, Luminance)表示明亮度，UV(Chrominance,Chroma)表示色度。色度的意思就是颜色和饱和度，明亮度就是亮度了。因为在计算机上的颜色是由明亮度和色度共同叠加来表示的。黑白视频就只有Y分量，YUV就是彩色视频了，为了兼容性，彩电也可以忽略UV，以此来兼容黑白电视机。

UV信号告诉了电视要偏移某象素的的颜色，而不改变其亮度。或者UV信号告诉了显示器使得某个颜色亮度依某个基准偏移。UV的值越高，代表该像素会有更饱和的颜色。

彩色电视最早的构想是使用RGB三原色来同时传输。这种设计方式是原来黑白带宽的3倍，在当时并不是很好的设计。RGB诉求于人眼对色彩的感应，YUV则着重于视觉对于亮度的敏感程度，Y代表的是亮度，UV代表的是彩度（相近于RGB），因此YUV的记录通常以Y:UV的格式呈现。

为了节省带宽起见，大多数的YUV格式平均使用的美味像素都少于24位，也就是少于3个字节。主要的采样格式有YUV4:2:0 YUV4:2:2 YUV4:1:1 YUV4:4:4。YUV的表示法称为A:B:C表示法。

所以YUV格式你可以看到，YUV420P, YUV422, YUV444, YUV411。

比如YUV444,就是 4:4:4，也就是完全采样，每个像素点就有YUV三个分量，和RGB一样，每个像素点占用3个字节。

![[Pasted image 20220704214914.png]]

YUV420P就是4:2:0，也就是每4个Y分量，共用一对UV分量。YUV420P和YUV420SP是不一样的，注意看下图:

![[Pasted image 20220704215058.png]]

![[Pasted image 20220704214738.png]]

YUV422就是4:2:2，就是每两个Y分量，共用一对UV分量。

![[Pasted image 20220704214859.png]]


[Jpeg图像 压缩/解码 之采样因子_weixin_33924220的博客-CSDN博客](https://blog.csdn.net/weixin_33924220/article/details/91974232)


注意，YUV大体分packed格式和planar格式，packed类似于RGB这样三个分量对应的一个像素是存储在一起的，而planar是三个分量分别存放在不同的地方，Y后紧跟随U和V这样的。

## YUV420P像素数据分离YUV分量

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
 
		fread(pic,1,w*h*3/2,fp); // fread读取了N个字节，文件指针就像后移动N字节
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


## YUV444P像素数据分离YUV分量

以下C代码提取YUV444P数据中的Y,U,V分量，分别保存成三个文件:

```cpp
/**
 * Split Y, U, V planes in YUV444P file.
 * @param url  Location of YUV file.
 * @param w    Width of Input YUV file.
 * @param h    Height of Input YUV file.
 * @param num  Number of frames to process.
 *
 */
int simplest_yuv444_split(char *url, int w, int h,int num){
	FILE *fp=fopen(url,"rb+");
	FILE *fp1=fopen("output_444_y.y","wb+");
	FILE *fp2=fopen("output_444_u.y","wb+");
	FILE *fp3=fopen("output_444_v.y","wb+");
	unsigned char *pic=(unsigned char *)malloc(w*h*3);
 
	for(int i=0;i<num;i++){
		fread(pic,1,w*h*3,fp);
		//Y
		fwrite(pic,1,w*h,fp1);
		//U
		fwrite(pic+w*h,1,w*h,fp2);
		//V
		fwrite(pic+w*h*2,1,w*h,fp3);
	}
 
	free(pic);
	fclose(fp);
	fclose(fp1);
	fclose(fp2);
	fclose(fp3);
 
	return 0;
}
```

调用如下:

```cpp
simplest_yuv444_split("lena_256x256_yuv444p.yuv",256,256,1);
```

从以上代码中可以看到YUV444P一帧像素数据占$w*h*3$个字节，其中前$w*h$个字节存储Y分量，接着$w*h$个字节存储U分量，最后的$w*h$存储V分量。

上述代码运行后，将会把一张分辨率为$256*256$的YUV444P文件分离为三个:

```txt
output_444_y.y：纯Y数据，分辨率为256x256。  
output_444_u.y：纯U数据，分辨率为256x256。  
output_444_v.y：纯V数据，分辨率为256x256。
```

![[Pasted image 20220704205028.png]]

输入原图

![[Pasted image 20220704205038.png]]

output_444_y.y

![[Pasted image 20220704205047.png]]

output_444_u.y

![[Pasted image 20220704205056.png]]
output_444_v.y


## YUV420P像素数据去掉颜色

也就是变成纯灰度图。

```cpp
/**
 * Convert YUV420P file to gray picture
 * @param url     Location of Input YUV file.
 * @param w       Width of Input YUV file.
 * @param h       Height of Input YUV file.
 * @param num     Number of frames to process.
 */
int simplest_yuv420_gray(char *url, int w, int h,int num){
	FILE *fp=fopen(url,"rb+");
	FILE *fp1=fopen("output_gray.yuv","wb+");
	unsigned char *pic=(unsigned char *)malloc(w*h*3/2);
 
	for(int i=0;i<num;i++){
		fread(pic,1,w*h*3/2,fp);
		//Gray
		memset(pic+w*h,128,w*h/2);
		fwrite(pic,1,w*h*3/2,fp1);
	}
 
	free(pic);
	fclose(fp);
	fclose(fp1);
	return 0;
}

simplest_yuv420_gray("lena_256x256_yuv420p.yuv",256,256,1);
```

从代码上可以看出，想把YUV420P变成灰度图，只需要将U,V分量全部设置成128就可以了。以上memset代码与以下代码等价:

```cpp
// U
memset(pic+w*h, 128, w*h/4);
// V
memset(pic+w*h*5/4, 128, w*h/4);
```

这是因为U、V是图像中的经过偏置处理的色度分量。色度分量在**偏置处理前**的取值范围是-128至127，这时候的无色对应的是“0”值。经过偏置后色度分量取值变成了0至255，因而此时的无色对应的就是128了。偏置处理相当于把取值范围的区间移动了。

![[Pasted image 20220704205807.png]]
输入原图

![[Pasted image 20220704205818.png]]
灰度处理后的图



为什么会有YUV420P, YUV444P等这些东西呢？它们有什么不同？那么请看

[图像原始格式(YUV444 YUV422 YUV420)一探究竟 - mcdull^0^ - 博客园 (cnblogs.com)](https://www.cnblogs.com/tid-think/p/10616789.html)

[YUV图解 （YUV444, YUV422, YUV420, YV12, NV12, NV21） - schips - 博客园 (cnblogs.com)](https://www.cnblogs.com/schips/p/12099550.html)

[[2 of 6] Comparison between H.264 YUV420 and YUV444 – GRID4ALL (sschaber.de)](http://sschaber.de/2018/12/03/2-of-6-comparison-between-h-264-yuv420-and-yuv444/)



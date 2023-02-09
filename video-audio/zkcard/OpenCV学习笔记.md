## OpenCV 摄像头 & 推流

 [C++ OpenCV 顯示camera攝影機串流影像 | ShengYu Talk (shengyu7697.github.io)](https://shengyu7697.github.io/opencv-camera/)
 [How to capture video from other cameras in OpenCV using C++? (tutorialspoint.com)](https://www.tutorialspoint.com/how-to-capture-video-from-other-cameras-in-opencv-using-cplusplus)

[How to capture video from default camera in OpenCV using C++? (tutorialspoint.com)](https://www.tutorialspoint.com/how-to-capture-video-from-default-camera-in-opencv-using-cplusplus)

[Play Video from File or Camera - OpenCV Tutorial C++ (opencv-srf.com)](https://www.opencv-srf.com/2017/12/play-video-from-file-or-camera.html)

[Opencv tutorial RTMP video streaming to NGINX restream as HLS (funvision.blogspot.com)](https://funvision.blogspot.com/2022/02/video-streaming-for-machine-learning.html)

[Multimedia: Which is better: FFmpeg or GStreamer? Why? - Quora](https://www.quora.com/Multimedia-Which-is-better-FFmpeg-or-GStreamer-Why)

[streaming - How to stream via RTMP using Gstreamer? - Stack Overflow](https://stackoverflow.com/questions/41550901/how-to-stream-via-rtmp-using-gstreamer)

[python 3.x - How to output x265 compressed video with cv2.VideoWriter - Stack Overflow](https://stackoverflow.com/questions/61260182/how-to-output-x265-compressed-video-with-cv2-videowriter/61281547#61281547)

[Pipe and OpenCV to FFmpeg with audio streaming RTMP in Python - Stack Overflow](https://stackoverflow.com/questions/70471732/pipe-and-opencv-to-ffmpeg-with-audio-streaming-rtmp-in-python/70476737#70476737)

[jkuri/opencv-ffmpeg-rtmp-stream: OpenCV FFMpeg Live Video Stream over RTMP protocol. (github.com)](https://github.com/jkuri/opencv-ffmpeg-rtmp-stream)

#### OpenCV人脸检测以及人脸识别


[Face Detection on recorded videos using OpenCV in Python — Windows and macOS | by Venkatesh Chandra | Analytics Vidhya | Medium](https://medium.com/analytics-vidhya/face-detection-on-recorded-videos-using-opencv-in-python-windows-and-macos-407635c699)


[OpenCV: Face Detection in Video Capture](https://docs.opencv.org/3.4/df/d6c/tutorial_js_face_detection_camera.html)

[(4条消息) Dlib模型实现人脸识别_浮屠tufu的博客-CSDN博客_dlib人脸识别](https://blog.csdn.net/weixin_46075497/article/details/121410047)

[利用dlib库实现人脸识别_枫叶的鱼的博客-CSDN博客_dlib人脸检测](https://blog.csdn.net/w798214705/article/details/121410894)

[(4条消息) 人脸识别——基于DLib库_Mr_Nobody17的博客-CSDN博客_人脸识别dlib库](https://blog.csdn.net/Mr_Nobody17/article/details/120352866)

[(4条消息) Python调用dlib库实现人脸识别 — AI初学者快速体验人工智能实现_周雄伟的博客-CSDN博客_dlib face model](https://blog.csdn.net/ebzxw/article/details/80441556)


#### 人脸检测与识别算法比较

主要是OpenCV的。介绍5种最流行的人脸检测识别算法。

1. OpenCV里的Haarcascade

这是一种基于机器学习的方法，从大量的正面和负面图像中训练出一个级联函数。然后，它可以被用于任何我们想要检测人脸的图像上。它以能够检测图像中的人脸和人脸部位而闻名，但也可以被训练来检测绝大多数的物体。

```python
import numpy as np 
import cv2 

#Load the haarcascade file 
cascPath = "/home/cale/.local/lib/python3.8/site-packages/cv2/data/haarcascade_frontalface_default.xml" faceCascade = cv2.CascadeClassifier(cascPath) 

cap = cv2.VideoCapture("data.mp4") 

while(True): 
	ret, frame = cap.read() 
	frame = cv2.resize(frame, (600, 400)) 
	faces = faceCascade.detectMultiScale2(frame, scaleFactor=1.1, minNeighbors=5, flags=cv2.CASCADE_SCALE_IMAGE) 
	for (x, y, w, h) in faces[0]: 
		conf = faces[1][0][0] 
		if conf > 5: 
			text = f"{conf*10:.2f}%" 
					cv2.putText(frame, text, (x, y-20),cv2.FONT_HERSHEY_SIMPLEX, 1,(170, 170, 170), 1) 
					cv2.rectangle(frame, (x, y), (x+w, y+h), (255, 255, 255), 1) 
					cv2.imshow("Frame", frame) 
					if cv2.waitKey(25) & 0xFF == ord('q'): 
						break 
						
cap.release() 
cv2.destroyAllWindows()
```

2. OpenCV的DNN(Deep Neural Network， 深度神经网络)

除了OpenCV的基于haarcascade滤波器的检测算法之外，OpenCV还发布了一个dnn模块，它代表深度神经网络。

3. Dlib

Dlib不是某种算法，它是一个非常有用和实用的工具包，用于制作现实世界的机器学习和数据分析应用程序。它是一个基于CNN(卷积神经网络)的检测器，一般来说，它能够从几乎所有角度检测人脸。我们树莓派上的检测程序就是调用了Dlib库的算法，只是选择了非CNN的算法，是HOG的特征提取算法。HOG和CNN的识别有什么不同呢？HOG相比CNN识别速度会更快，但是精准度相比CNN会低一些，CNN识别准确度会更高。

```python
import dlib 
import cv2 #We create the model here with the weights placed as parameters 
face_detect = dlib.cnn_face_detection_model_v1("/content/mmod_human_face_detector.dat") 
cap = cv2.VideoCapture("data.mp4") 

while True: 
	ret, frame = cap.read() 
	frame = cv2.resize(frame, (600, 400)) 
	faces = face_detect(frame, 1) 
	
	for face in faces: 
		# In dlib in order to extract points we need to do this 
		x1 = face.rect.left() 
		y1 = face.rect.bottom() 
		x2 = face.rect.right() 
		y2 = face.rect.top() 
		cv2.rectangle(frame, (x1, y1), (x2, y2), (255, 0, 0), 1) 
	
	cv2.imshow("Frame", frame) 
	if cv2.waitKey(25) & 0xFF == ord('q'): 
		break 
cap.release()
cv2.destroyAllWindows()
```

4. 多任务层叠卷积神经网络(MTCNN, multi-task cascaded Convolutional Neural Network)

MTCNN或多任务级联卷积神经网络无疑是当今最流行和最准确的人脸检测工具之一。因此，它基于深度学习架构，具体由3个神经网络（P-Net、R-Net和O-Net）组成，以级联方式连接。

```python
#You can install mtcnn using PIP by typing "pip install mtcnn" 
from mtcnn import MTCNN 
import cv2 
detector = MTCNN() 
#Load a video, if we were using google colab we would 
#need to upload the video to Google Colab 
cap = cv2.VideoCapture("data.mp4") 

while(True): 
	ret, frame = cap.read() 
	frame = cv2.resize(frame, (600, 400)) 
	boxes = detector.detect_faces(frame) 
	if boxes: 
		box = boxes[0]['box'] 
		conf = boxes[0]['confidence'] 
		x, y, w, h = box[0], box[1], box[2], box[3] 
		
		if conf > 0.5: 
			cv2.rectangle(frame, (x, y), (x+w, y+h), (255, 255, 255), 1) 
	cv2.imshow("Frame", frame) 
	if cv2.waitKey(25) & 0xFF == ord('q'): 
		break 
cap.release() 
cv2.destroyAllWindows()
```

5. 用Facenet提供的算法

Facenet是一个人脸检测系统，可以说是一个统一的人脸检测和聚类的嵌入。它是一个系统，当给定一张人脸图片时，它将从人脸中提取高质量的特征。这个128个元素的向量被用于未来预测和检测人脸，它一般被称为人脸嵌入。

这个模型是一个深度卷积神经网络，使用三重损失函数进行训练。它鼓励相同身份的向量变得更加相似，而不同身份的向量则有望变得不太相似。

训练模型的重点是直接创建嵌入，而不是从模型的中间层提取。此外，这也是这个算法中的创新。

```python
#You can install facenet using PIP by typing "pip install facenet-pytorch" 
#Import the required modules 
from facenet_pytorch import MTCNN 
import torch 
import cv2 

device = torch.device('cuda:0' if torch.cuda.is_available() else 'cpu') 

#Create the model 
mtcnn = MTCNN(keep_all=True, device=device) 

#Load the video and go from frame to frame 
cap = cv2.VideoCapture("data.mp4") 

while True: 
	ret, frame = cap.read() 
	if ret: 
		frame = cv2.resize(frame, (600, 400)) 
		#Here we are going to use the facenet detector 
		boxes, conf = mtcnn.detect(frame) 
		# If there is no confidence that in the frame is a face, don't draw a rectangle around it 
		if conf[0] != None: 
			for (x, y, w, h) in boxes: 
				text = f"{conf[0]*100:.2f}%" 
				x, y, w, h = int(x), int(y), int(w), int(h)
				cv2.putText(frame, text, (x, y-20), cv2.FONT_HERSHEY_SIMPLEX, 1,(170, 170, 170), 1)
				cv2.rectangle(frame, (x, y), (w, h), (255, 255, 255), 1) 
	else: 
		break 
#Show the result 
#If we were using Google Colab we would use their function cv2_imshow() 
#For displaying images/frames 
	cv2.imshow("Frame", frame) 
	if cv2.waitKey(25) & 0xFF == ord('q'): 
		break 
cap.release() 
cv2.destroyAllWindows()
```

接下来对这5种算法进行总结比较

以上算法我们都对人脸进行了方框框住了，就精准度来说，测试了几个视频，肉眼几乎没有发现识别出错的图像，也就是说，在要求不是很极端的场景下，这几种算法在精准度无需下功夫选择，接下来就是比较性能和速度了。下面有一张图:

![[Pasted image 20220809100858.png]]


由上图可知，dlib平均处理每帧图像所需要的时间最短。MTCNN是最长的。


#### 检测人脸对比

```python
#!/usr/bin/env python3

# pip install face_recognition -i https://mirrors.aliyun.com/pypi/simple/

import subprocess as sp
import sys
import cv2
import face_recognition
from multiprocessing import Process
import time
from mtcnn import MTCNN
from yoloface import face_analysis
import numpy
from pathlib import Path

algo_name = ""
input_file = ""

def main():
    if algo_name == "" or input_file == "":
        print("Please set algo_name and input file")
        sys.exit(1)
    
    print("entry %s service" % algo_name)

    if algo_name == "hog":
        image = face_recognition.load_image_file(input_file)
        bgr_image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)
        face_locations = face_recognition.face_locations(image)

        for top, right, bottom, left in face_locations:
            # Draw a box around the face
            cv2.rectangle(bgr_image, (left, top), (right, bottom), (0, 0, 255), 2)
        
        filename = "output_%s_%s.jpg" % (algo_name, Path(input_file).stem)
        cv2.imwrite(filename, bgr_image)
    
    elif algo_name == "cnn":
        image = face_recognition.load_image_file(input_file)
        bgr_image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)
        face_locations = face_recognition.face_locations(image, 1, model="cnn")

        for top, right, bottom, left in face_locations:
            # Draw a box around the face
            cv2.rectangle(bgr_image, (left, top), (right, bottom), (0, 0, 255), 2)
        
        filename = "output_%s_%s.jpg" % (algo_name,  Path(input_file).stem)
        cv2.imwrite(filename, bgr_image)
    elif algo_name == "haar":
        casc_path = "/home/mathxh/.local/lib/python3.8/site-packages/cv2/data/haarcascade_frontalface_default.xml"
        faceCascade = cv2.CascadeClassifier(casc_path)
       
        image = face_recognition.load_image_file(input_file)
        bgr_image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)

        face_locations = faceCascade.detectMultiScale2(bgr_image, scaleFactor=1.1, minNeighbors=5, flags=cv2.CASCADE_SCALE_IMAGE)
        for x, y, w, h in face_locations[0]:
            # Draw a box around the face
            cv2.rectangle(bgr_image, (x, y), (x+w, y+h), (0, 0, 255), 2)

        filename = "output_%s_%s.jpg" % (algo_name,  Path(input_file).stem)
        cv2.imwrite(filename, bgr_image)
    elif algo_name == "mtcnn":
        image = face_recognition.load_image_file(input_file)
        bgr_image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)

        detector = MTCNN()
        results = detector.detect_faces(bgr_image)
        if results:
            box = results[0]['box']
            conf = results[0]['confidence']
            x, y, w, h = box[0], box[1], box[2], box[3]
            if conf > 0.5:
                cv2.rectangle(bgr_image, (x, y), (x+w, y+h), (0, 0, 255), 2)
        
        filename = "output_%s_%s.jpg" % (algo_name,  Path(input_file).stem)
        cv2.imwrite(filename, bgr_image)
    elif algo_name == "yolo":
        image = face_recognition.load_image_file(input_file)
        bgr_image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)

        face = face_analysis()
        img,box,conf=face.face_detection(image_path=input_file,model='tiny')
        output_frame = face.show_output(img,box,frame_status=True)
        filename = "output_%s_%s.jpg" % (algo_name,  Path(input_file).stem)
        cv2.imwrite(filename, output_frame)

if __name__ == "__main__":
    algo_name = sys.argv[1]
    input_file = sys.argv[2]
    main()



```

##### 识别人脸

```python
#!/usr/bin/env python3

# pip install face_recognition -i https://mirrors.aliyun.com/pypi/simple/

import subprocess as sp
import sys
import cv2
import face_recognition
from multiprocessing import Process
import time
from mtcnn import MTCNN
from yoloface import face_analysis
import numpy as np
from pathlib import Path
from PIL import Image, ImageDraw, ImageFont

input_file = ""

def cv2AddChineseText(img, text, position, textColor=(0, 255, 0), textSize=30):
    if (isinstance(img, np.ndarray)):  # 判断是否OpenCV图片类型
        img = Image.fromarray(cv2.cvtColor(img, cv2.COLOR_BGR2RGB))
    # 创建一个可以在给定图像上绘图的对象
    draw = ImageDraw.Draw(img)
    # 字体的格式 simsun.tcc是中文字体文件
    fontStyle = ImageFont.truetype(
        "simsun.ttc", textSize, encoding="utf-8")
    # 绘制文本
    draw.text(position, text, textColor, font=fontStyle)
    # 转换回OpenCV格式
    return cv2.cvtColor(np.asarray(img), cv2.COLOR_RGB2BGR)


def main():
    if input_file == "":
        print("Please set  input file")
        sys.exit(1)

    id_card_image = face_recognition.load_image_file('id_card.jpg')
    id_card_face_encoding = face_recognition.face_encodings(id_card_image)[0]

    known_face_encodings = [id_card_face_encoding]
    known_face_names = ['陈绍涵']

    image = face_recognition.load_image_file(input_file)
    bgr_image = cv2.cvtColor(image, cv2.COLOR_RGB2BGR)
    face_locations = face_recognition.face_locations(image)
    face_encodings = face_recognition.face_encodings(image, face_locations)

    face_names = []
    for face_encoding in face_encodings:
        matches = face_recognition.compare_faces(known_face_encodings, face_encoding)
        name = "Unknown"
    
        face_distances = face_recognition.face_distance(known_face_encodings, face_encoding)
        best_match_index = np.argmin(face_distances)
        if matches[best_match_index]:
            name = known_face_names[best_match_index]

        face_names.append(name)

    for (top, right, bottom, left), name in zip(face_locations, face_names):
        # Draw a box around the face
        cv2.rectangle(bgr_image, (left, top), (right, bottom), (0, 0, 255), 2)
        print("sss: %s" % name)
        # Draw a label with a name below the face
        # cv2.rectangle(bgr_image, (left, bottom - 35), (right, bottom), (0, 0, 255), cv2.FILLED)
        # font = cv2.FONT_HERSHEY_DUPLEX
        # cv2.putText(bgr_image, name, (left + 6, bottom + 15), font, 5.0, (0, 0, 255), 2)
        bgr_image = cv2AddChineseText(bgr_image, name, (left + 6, bottom + 15), (255,0,0), 100)
    filename = "output_%s_%s.jpg" % ('face_recg', Path(input_file).stem)
    cv2.imwrite(filename, bgr_image)
    
    

if __name__ == "__main__":
    input_file = sys.argv[1]
    main()

```
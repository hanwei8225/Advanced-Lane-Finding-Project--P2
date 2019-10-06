

**Advanced Lane Finding Project**

本项目分为以下几步：

* 通过一组棋盘图像计算出镜头的标定矩阵和畸变系数
* 使用计算出的数据对原始图像进行校正
* 使用梯度和颜色等对图像进行二值化的变化。
* 使用ROI对图像进行裁剪
* 使用透视原理将图像投影为鸟瞰图
* 检测车道上的像素，并且对其进行多项式拟合，找到边界
* 检测车道的曲率以及车中心相对于车道中心的位置
* 将车道内部使用颜色标出，并且反投影回原图像方向
* 可视化的呈现以及标出曲率和车头偏移量

[//]: # (Image References)

[image1]: ./output_images/usdistorted_image.png
[image2]: ./output_images/abs_sobel_x_y.png
[image3]: ./output_images/mag_and_dir.png
[image4]: ./output_images/combined_gradient.png
[image5]: ./output_images/hls_and_bgr.png
[image6]: ./output_images/combined_color.png
[image7]: ./output_images/combined.png
[image8]: ./output_images/masked.png
[image9]: ./output_images/warped.png
[image10]: ./output_images/field.png
[image11]: ./output_images/back_recitify.png
[image12]: ./output_images/mixed.png
[image13]: ./output_images/result.png
[image14]: ./output_images/fit.png


[video1]: ./project_video.mp4 
[video2]: ./output_images/project_video.mp4
[video3]: ./output_images/project_video_outing.mp4
[video4]: ./output_images/project_video_v2.mp4
[video5]: ./output_images/project_video_v2.mp4



## 以下是每一个要点的实现思路和结果，所有的代码均在P2.ipynb中，并且均有注释和相应的标题做区分

---



### 镜头校准

#### 1. 原理
镜头校准的主要原理是通过一组棋盘图（存放在/camera_cal中），获取到该镜头的校准矩阵mtx和畸变系数dist，通过这两个数据，就可以校准镜头采集到的图像。其中要注意的点是：
1.1 读取的时候是读取整组图片。
1.2 需要设置棋盘图的黑白交叉点的数量，这个数量为一个元组，（行，列）。
#### 结果

![alt text][image1]

### 图像二值化

#### 1.梯度
使用梯度对图像进行二值化，使用的是灰度化图像后，图像在某一个区域内颜色深度的变化率来检测边缘，从而将图片内物体的边缘描绘出来，我们里使用的是索贝尔算子（sobel）进行梯度监测，在这里，我使用了x轴方向的绝对梯度，y轴方向的绝对梯度，x和y方向的合成梯度，以及梯度的角度来设置阈值，分别求出每一个特征下的二值化图像，然后进行布尔运算，获得一个相对来说比较干净的边缘图。
以下是结果：
##### x和y方向的绝对梯度作为阈值
![alt text][image2]
##### 合成梯度和角度作为阈值
![alt text][image3]
##### 梯度布尔运算组合图
![alt text][image4]

#### 2.色彩空间
在常用的RGB色彩空间中，黄色车道线不能被很好的检测出来，尤其是一些污损的和褪色的车道线，这就需要使用其他的色彩空间，在这里我使用的是HLS色彩空间，针对黄色车道线进行提取，经过试验发现，在HLS空间中的S通道上对黄色车道的检测较为清晰，而在RGB色彩空间中R通道对白色车道检测清晰，因此需要分别做检测然后合并
##### HLS中的S通道和RGB中的R通道的检测结果
![alt text][image5]
##### 两种色彩空间二值化图像组合
![alt text][image6]
#### 3.梯度和图像二值化图片的布尔组合
![alt text][image7]


### 使用RIO对图像进行裁剪
对于整个图像来说，我们感兴趣的只有其中一部分图像，其余的部分为已知的无用图像，为了避免这部分图像对我们的计算产生影响，我们使用一个遮罩来屏蔽掉一些无用的部分，例如天空等，在这里我使用的是一个7边型进行裁剪，裁剪过后的图像如图
![alt text][image8]


### 投影
因为我们目前的图像是从斜上方拍摄，根据投影的规律，越往远的图像就会变得越小，这样很难计算出车道的曲率，所以我们要使用投影定理将斜上方拍摄的图像投影成鸟瞰图。
在这里我们需要获取原始图像上的四个点和投影变换后对应的四个点的坐标，以此来完成投影。
我首先使用了霍夫空间拟合车道线，并且截取适当的距离获得了投影前的四个点：
```python
src = np.float32([[ 205,720],[ 565, 470],[715,470],[1107,720]])
```
随后经过多次尝试，获得了投影后对应的四个点的位置：
```python
dst = np.float32([[ 205,720],[ 205,235],[1100,235],[1100,720]])

```
经过几次对src和dist的微调，最终确定的坐标为：

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 205, 720      | 300, 720        | 
| 565, 470      | 300, 235      |
| 715, 470     | 980, 235      |
| 1107, 720      | 980, 720        |

使用这两组点，可以计算出投影矩阵M和逆投影矩阵M_R，作为图像投影的关键参数。
以下为投影变换过后的图像：
![alt text][image9]



### 拟合左右车道线的多项式
使用一个非0像素的直方图统计，可以获得在X轴上像素最多的点，这个点就是左右车道的中心线，然后以这个中心线为基准，使用一个小窗口依次向上扫描，圈住最多的非0像素点，即可将整条车道线圈出，然后使用圈出的像素坐标分别拟合一个二次曲线，即为左右车道线的中心：
![alt text][image14]


### 计算车道曲率和车头偏离中心线距离
通过公式计算车道的曲率，公式具体解释在这里：https://www.intmath.com/applications-differentiation/8-radius-curvature.php，在python中的实现方法为：

```python
    road_left = (left_fit[0] * (y_eval ** 2)) + (left_fit[1] * y_eval) + left_fit[2]
    road_right = (right_fit[0] * (y_eval ** 2)) + (right_fit[1] * y_eval) + right_fit[2]
   
```
其中fit为拟合出的二次曲线的三个系数，y_eval为y的某一点坐标值，这里取做靠近车的一点

考虑到像素和实际距离的对应关系，对算法进行了如下改变
```python
    ym_per_pix = 30/720
    xm_per_pix = 3.7/700
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval * ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) /np.absolute(2*right_fit_cr[0])

```
同时可以计算出相机的中心（默认相机安装在汽车中心）和车道线的中心的差值，然后根据比例换算为距离

### 将车道使用颜色覆盖

在找出车道后，我们使用绿色填充整个区域，如图：
![alt text][image10]

### 将覆盖层反变换为原图视角
因为我们所有的覆盖均为在鸟瞰图上完成，所以需要将鸟瞰图反变换为正常视角，使用之前求出的M_R进行反变换：
![alt text][image11]


### 将覆盖层和原图合并
因为覆盖层只有我们提取出来的信息，为了方便呈现，我们需要将覆盖层半透明的覆盖到原图上：
![alt text][image12]

### 将曲率和距离显示
我们需要将曲率和车头偏离距离显示到图片上：
![alt text][image13]


至此，对于单张图片的处理就完成，之后会对视频进行处理，视频的处理可以等效为对多张图片的处理


## 视频的处理

### Pipeline (video)

在这里将上述对图片的处理进行了一个整合，详细见P2.ipynb中的代码

### 第一次处理

![alt text][video2]
可以看到在复杂的地形检测车道会出现问题，经过排查，发现是做投影的时候把车道投影太宽了，导致弯道的时候会有一侧的车道线超出屏幕，在调整了投影参数后，进行了第二次处理：

### 第二次处理
![alt text][video4]

第二次处理的结果比第一次好不少，但是在有一些干扰的情况下还是会出现个别帧检测失败的问题。

### 第三次处理
这次进行了一些尝试，之前的检测失败的问题的产生原因是车道线有一侧是断掉的，有时候像素过少就会拟合不准确。
但是我们知道车道线是平行的，因此我选择检测到的像素更多的一侧作为基准，另一侧只是将基准侧进行平移后得到，进行测试后相对较好，但是也有可能会产生更多的未知问题
![alt text][video4]

---

## 讨论
检测失败的地方主要是车道线磨损，或者车道线断裂导致的检测到的像素过少，也有一部分是因为有一些道路颜色切换带来的干扰项，所以提升的地方我认为主要有以下几个方面：
1.更为可靠的检测磨损和各种光线条件下的车道线的算法
2.排除道路颜色切换带来的干扰
3.使用我第三次处理视频的方法，只要有一条车道线边界是可靠的，就可以使用平移来拟合出两条车道线。


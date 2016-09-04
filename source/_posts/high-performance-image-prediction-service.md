---
title: Build a high performance image prediction service
date: 2016-09-04 22:36:00
tags:
---

Deep learning is absolutely one of the keywords in these years. Like many other company, we use deep learning to solve some tasks, like image classification, OCR, face detection, etc. I’ve  spent some time building the image classification service and done some optimizations during my work.  I will share some of my experience in this article.

<!--more-->

### CPU or GPU

It really depends on a variety of factors, such as the request scale, power consumption, etc. GPU is expensive, but running the forward phase on the GPU device has dozens of times speed-up than CPU even you use OpenBLAS or MKL to accelerate the underlying matrix manipulation in CPU.

### Prediction Model design

In general case, people will try to predict the image category using convolutional neural networks (CNN), like AlexNet, GoogleNet, or VGGNet. Different networks has different prediction performance and time complexity in inference phase. For example, executing forward stage in AlexNet is 4 times faster than GoogleNet. But GoogleNet has better classification ability since it’s deeper and wider. Directly training the pre-trained model on the ImageNet or some other open database might not meet your need for precision and recall. To gain a higher precision and recall, there are many approaches worth to try: re-design the network, use the cascaded model, integrate the outputs of multi-network, predict with multiple scales. All these strategies are doable and sound great. But remember, these designs might more or less increase the time and GPU memory cost. So, take the time and space complexity into account while designing the neural network models.

### Prediction Frameworks
Building an neural network prediction framework from scratch of you own might not a smart idea since there are some well-built projects such as Caffe, MXNet and TensorFlow, which can be deployed directly in the production environment. Take following pros and cons into account before choosing one.
- Caffe. Maybe this is the most popular framework that people are familiar with, which means you can find lots of solutions for the problem you encounter on the Internet. Caffe has graceful C++ and python API. The cons is that it occupies too much GPU memory when comparing with MXNet and TensorFlow. This is the design issue. Caffe will allocate the memory for parameters and output of each layer and these memory can not be reused. For GoogleNet model, Caffe allocates 6x GPU memory than MXNet. The good news is that if you want to load multiple identical models on the same process, you can share the parameters of each layer.
- MXNet. A deep learning that has a better design of GPU memory allocation. It constructs the neural network as a computation graph. Each node in the graph can be reuse and it has high performance since the computation is not executed layer-by-layer, but based on the node (a node is a computation unit like add, minus, multiply and divide). Concretely, a node can be executed if its preceding nodes are finished. Like Caffe, MXNet also has graceful C++ and python API. The bad news is MXNet has worse performance when running on CPU. Take a look of this issue.
- TensorFlow. The GPU memory usage and performance are close to MXNet. I don’t have experience on TensorFlow on production environment. The official team provides a project, called serving, to help deploy deep learning models.

### Single or Batch prediction
Running a GoogleNet V1 in forward (with cuDNN) with the batch size of 10 is only~30% time of running batch size of 1 in 10 times. This conclusion provides us a way to improve the service performance. We can add a middleware to serve the request from client and forward them to prediction service. There are some features for the middleware showing as followings:
- Request coalescing duration. The middleware needs to forward the batched images within a limited period of time even the requested imaged to be predicted is smaller than the batch size. If not, it brings high-latency. 
- Batch size. In general, the throughput of the prediction system will be enhanced if batch size increased. But too larger a batch size leads to high GPU memory occupation and service latency.
- Rate limiting. The middleware can limit the number of request to fit the service limitation of prediction service.

Following figure a flexible and extensible architecture.

### Image decode optimization
Decode a 1080P image will cost ~30ms in CPU while running inference of GoogleNet V1 only cost ~10ms in GPU. Thus, decoding image may somehow become bottleneck. To solve this, It’s better to resize the image in front end. In addition, decoding image with in libjpeg-turbo library ~30% faster than libjpeg (I got this result but the official site claims that better performance can be achieved).

### Conclusion
The strategies above mentioned are not only suitable for image prediction service, but also for face recognition, face detection, etc.

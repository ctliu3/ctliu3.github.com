title: Tips you should know about Caffe
date: 2015-08-08 16:49:18
tags: [ML, DL, Caffe]
---

[Caffe][1] is a wonderful deep learning (DL) framework, written in C++ and it is still under construction. I use Caffe to do some image related tasks these days. Technologically, I use Caffe just around one week, so I’m also a newbie. I encounter some “pits” when using Caffe. So I write down some tips, hope they are helpful to you.

<!--more-->

### What Caffe can do?
Caffe is actually a feature learning tool, you can do some specific tasks like classification, clustering, segmentation with the help of it. You can define the structure of the network to train the model of your own. Otherwise, some pre-trained models, including [googlenet][10], [caffenet][11], are provided, they can be used directly. 

### Read the source code
In some degree, Caffe is really a blackbox. Therefore, if you first time meet it, you may want to dig into the source code to see the scenery behind Caffe. Knowing the following modules, which are described in a high level, will make your travel easier.

- [Blob][2]. The data that flows up and down in the deep learning network is wrapped by Blob class. Blob is a 4-dimensional (4D) data and it contains different kinds of information when it is in different positions in the net. For the input, the shape of data is `#image x #channel x height x width`. For the output, it is `#image x #class x 1 x 1`. Thus, you can compare the `#class` predictions and choose the maximal value to get which class each image belongs to.
- [Net][3]. This module describes the DL net. You will find many variables in the [net.cpp][3], but don’t be dismay, many of them are used to keep the structure of DL net. Think about how you construct a [circled acyclic graph][13] (DAG) in C++. It’s almost the same. But you do really need to know the input and output data of the net, as described above.
- [Layer][4]. This is a significant module in Caffe. One thing you need to know is, for a layer, there may be more than one layers under it and more than one layer above it. That’s it. If you are not a researcher, you can skip this amount of implements of different layers, which make one petrified. But I do recommend you should understand the function of each layer by their layer types.

Caffe uses [protocol buffer][9] to define the data format, so reading the [caffe.proto][12] file will be your first step before reading other parts of the source code.

### Obtain the features extracted by DL net
Since DL net is a multiple layers network, you can extract the features generated in some layers to do the future task, such as clustering. Although you can extract any layer in the network, usually the last layer before output is a better option. The extracted features are stored in `leveldb` or `lmdb` format and you need to write code to obtain the plain data of feature data. There is a simple script written in python.

<pre>
import lmdb
from caffe.proto import caffe_pb2

def save2txt(db_name):
    img_db = lmdb.open(db_name)
    txn = img_db.begin()
    cursor = txn.cursor()
    cursor.iternext()

    datum = caffe_pb2.Datum()
    for key, value in cursor:
        datum.ParseFromString(value)
        data = caffe.io.datum_to_array(datum)
        data = np.reshape(data, (1, np.product(data.shape)))[0]
       // data is the feature with shape (#feat, ) 
       // Your code.
</pre>

### What’s more
During the usage of Caffe, you need to do lots of trivial work, like putting the data in certain directory, checking the output of each step is whether correct or not, or sometimes you need to visualize the output data in website in that you probably work on the remote machine, etc. 
In light of this, being familiar with [bash/sh][5], [python][6] or [ruby][7] will greatly improve your working efficiency. You can check my [code snippet][8] in github for fast learning of bash.

[1]: https://github.com/BVLC/caffe
[2]: https://github.com/BVLC/caffe/blob/master/src/caffe/blob.cpp
[3]: https://github.com/BVLC/caffe/blob/master/src/caffe/net.cpp
[4]: https://github.com/BVLC/caffe/blob/master/src/caffe/layer_factory.cpp
[5]: https://en.wikipedia.org/wiki/Bash_(Unix_shell)
[6]: https://www.python.org
[7]: https://www.ruby-lang.org/zh_cn/
[8]: https://gist.github.com/ctliu3/fc1148da3922cff09eb7
[9]: https://developers.google.com/protocol-buffers/
[10]: https://github.com/BVLC/caffe/tree/master/models/bvlc_googlenet
[11]: https://github.com/BVLC/caffe/tree/master/models/bvlc_reference_caffenet
[12]: https://github.com/BVLC/caffe/blob/master/src/caffe/proto/caffe.proto
[13]: https://en.wikipedia.org/wiki/Directed_acyclic_graph
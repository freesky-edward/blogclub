# TensorFlow Introduction(1)


## Introduction

TensorFlow is an open source software library for numerical computation using
data flow graphs. The graph nodes represent mathematical operations, while the
graph edges represent the multidimensional data arrays (tensors) that flow between
them. This flexible architecture lets you deploy computation to one or more CPUs or
GPUs in a desktop, server, or mobile device without rewriting code. TensorFlow also
includes TensorBoard, a data visualization toolkit.

### Environment

You must choose one of the following types of TensorFlow to install:

TensorFlow with CPU support only. If your system does not have a NVIDIA® GPU,
you must install this version. Note that this version of TensorFlow is typically
much easier to install (typically, in 5 or 10 minutes), so even if you have an
NVIDIA GPU, we recommend installing this version first.

TensorFlow with GPU support. TensorFlow programs typically run significantly faster
on a GPU than on a CPU. Therefore, if your system has a NVIDIA GPU meeting the
prerequisites shown below and you need to run performance-critical applications,
you should ultimately install this version.

We only install TensorFlow with CPU support in this blog

### Install anaconda

https://www.anaconda.com/download/#windows

### Configure env variable
Add C:\ProgramData\Anaconda3\Scripts and C:\ProgramData\Anaconda3\ in path
restart your computer

### Install pip
```
python get-pip.py
https://pip.pypa.io/en/stable/installing/#do-i-need-to-install-pip
```

### Varify the python and pip version
The tensorflow only support python3.5 and python3.6
```
C:\Users\>python
Python 3.6.3 |Anaconda, Inc.| (default, Oct 15 2017, 03:27:45) [MSC v.1900 64 bi
t (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

The tensorflow only support >= 8.0.1 pip version
```
C:\ProgramData\Anaconda3\Scripts>pip.exe -V
pip 9.0.1 from C:\ProgramData\Anaconda3\lib\site-packages (python 3.6)
```

### Install TensorFlow
run the following command in your cmd
```
pip3 install --upgrade tensorflow
```

The tensorflow installed successfully when you saw the following echo:
```
Successfully installed bleach-1.5.0 enum34-1.1.6 html5lib-0.9999999 markdown-2.6
.10 protobuf-3.5.0.post1 setuptools-38.2.4 tensorflow-1.4.0 tensorflow-tensorboa
rd-0.4.0rc3 werkzeug-0.13 wheel-0.30.0
```

### Configure tensorflow in pycharm
1.Create a Conda environment for tensorflow
```
C:> conda create -n tensorflow
```
2.Enable this Conda environment
```
C:> activate tensorflow
```

3. Copy python.exe that under the python36 to Conda envs
   folder (../Anaconda3/envs)
4. Create a python project, and choose c:\pROGRAMdATA\Anaconda\envs\pythong.exe
   as your Interpreter

There is another way to create the python project.
https://www.youtube.com/watch?v=83vR1Nz3dHA

## Write code in pycharm
There is a simplest tensorflow example about hello world in
[github](https://github.com/aymericdamien/TensorFlow-Examples/blob/master/examples/1_Introduction/helloworld.py)

```
import tensorflow as tf

# Simple hello world using TensorFlow

# Create a Constant op
# The op is added as a node to the default graph.
#
# The value returned by the constructor represents the output
# of the Constant op.
hello = tf.constant('Hello, TensorFlow!')

# Start tf session
sess = tf.Session()

# Run the op
print(sess.run(hello))
```

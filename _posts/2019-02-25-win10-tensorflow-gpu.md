# win10搭建tensorflow_gpu环境遇到的问题

###问题：

使用的软件版本： 

```
*  tensorflow-gpu 1.12.0  
*  cuda toolkit 10.0.130  
*  cudnn 7.4.1.5  
```

import tensorflow时报错：

```
ImportError: DLL load failed: The specified module could not be found.
```

###问题原因：
tensorflow 1.12.0版本不支持cuda 10。


###解决方案：

1. 安装nightly build 1.13 tensorflow版本。
2. 安装cuda 9版本。
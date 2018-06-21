---
title: R配置Keras
date: 2018-06-21 19:18:18
tags:
    - R
    - Keras
    - 深度学习
categories: 工具使用
---

[TensorFlow](https://www.tensorflow.org/)基本上成为了深度学习的标配。但是为了学习主要的内容（忽略底层实现细节）或者快速实现一个神经网络模型，[Keras](https://keras.io/)是一个很好的选择。

[Keras中文版文档](http://keras-cn.readthedocs.io/en/latest/)

## 环境准备

- [R](https://www.r-project.org/)
- [Rtools](https://cran.r-project.org/bin/windows/Rtools/)
- [Anaconda](https://www.anaconda.com/)

## 安装步骤

安装`R`的`Keras`需要用到`devtools::install_github()`命令，因此需要安装`devtools`包。

但是笔者用的`Windows 10`，直接安装包不会报错，但是使用该包下的命令会出一些奇怪的错误，例如在装`R-Keras`时就报超时。

所以建议在`Windows`下使用该包时，先去安装[Rtools](https://cran.r-project.org/bin/windows/Rtools/)。

同时需要安装好`Anaconda`，不然在后续步骤中会报错。

安装好`Rtools`之后，打开`R命令行`，在其中再安装一遍`devtools`包：

``` R
install.packages("devtools")
```

然后安装`R`的`Keras`接口：

``` R
devtools::install_github("rstudio/keras")
```

在命令行或者第一次运行`Keras`的脚本下输入：

``` R
library(keras)
install_keras()
```

然后会调用`Anaconda`为`R-Keras`建立一个环境，成功初始化，在以后的`R脚本`就不需要再使用`install_keras()`命令了。

## 安装GPU支持

事实上这部分操作是属于`Anaconda`那边的操作，毕竟`R-Keras`用的还是`Python`的`TensorFlow`后端。

[中文文档里面](http://keras-cn.readthedocs.io/en/latest/for_beginners/keras_windows/)就已经很详细地介绍了如何准备`GPU`环境：

- [CUDA](https://developer.nvidia.com/cuda-downloads)
- [cuDNN](https://developer.nvidia.com/cudnn)

文档写的时候上述Keras尚未支持高版本的`CUDA`和`cuDNN`，经笔者测试，是可以支持最新版的`CUDA`和`cuDNN`的。

安装好后，打开`Anaconda Navigator`管理软件，或者命令行模式：

``` bash
$ activate r-tensorflow
```

切换到`r-tensorflow`环境，安装`gpu`版`Keras`：

``` bash
$ conda install keras-gpu
```

等待安装过程（可能因为墙导致下载出问题，要么上梯子，要么辛苦一点重试几次安装命令吧）后，就可以开始使用了。

## 测试R-Keras

官网给了`MNIST`测试集作为Demo，然而`AWS`被墙了，整个Demo直接卡在下载数据集步骤就挂了。

笔者写了个简单的脚本用于测试：

``` R
library(keras)
library(ggplot2)

script_dir <- dirname(sys.frame(1)$ofile)
setwd(script_dir)

# 产生sample.size * num_props的矩阵， 默认80%数据作为训练集，20%作为验证集
gensam <- function(sample.size = 10000, num_props = 30, val_portion = 0.2)
{
    all.x <- array(dim = c(sample.size, num_props))

    all.y <- array(dim = c(sample.size, 1))

    for(i in (1 : sample.size))
    {
        all.x[i, ] <- array(round(runif(num_props, 0, 9)))
        if(sum(all.x[i, ]) >= 100)
        {
            all.y[i, 1] <- 1
        }
        else
        {
            all.y[i, 1] <- 0
        }
    }

    split_point <- (1 - val_portion) * sample.size

    list(
        train = list(
            x = all.x[c(1 : split_point), ],
            y = all.y[c(1 : split_point)]
        ),
        test = list(
            x = all.x[c((split_point + 1) : sample.size), ],
            y = all.y[c((split_point + 1) : sample.size)]
        )
    )
}

# 产生一个数据集
dataset <- gensam()

# 准备训练一个LSTM网络，重塑数据集变成维度为(样本数量，序列长度，序列元素的长度)
x_train <- array_reshape(dataset$train$x, c(nrow(dataset$train$x), 30, 1))
y_train <- dataset$train$y

x_test <- array_reshape(dataset$test$x, c(nrow(dataset$test$x), 30, 1))
y_test <- dataset$test$y

model <- keras_model_sequential()

model %>%
    layer_lstm(units = 200, activation = "hard_sigmoid", input_shape = c(30, 1)) %>%
    layer_dense(units = 1, activation = "sigmoid")

# 编译模型
model %>% compile(
  loss = 'binary_crossentropy',
  # 使用ADAM优化
  optimizer = optimizer_adam(lr = 0.001, beta_1 = 0.001, beta_2 = 0.999, epsilon = 10e-8),
  metrics = c('accuracy')
)

# 训练模型并记录
history <- model %>% fit(
  x_train, y_train,
  epochs = 10, batch_size = 100
)

eval_report <- model %>% evaluate(
  x_test, y_test
)

# 输出训练完的模型在验证集的情况
print(eval_report)

plt.df <- data.frame(
  x = c(1:length(history$metrics$loss)),
  loss = history$metrics$loss,
  acc = history$metrics$acc
)

# 输出LOSS和ACC在迭代中的变化图形
ggplot(plt.df, aes(x = x)) + geom_point(aes(y = loss, color = "Loss")) + geom_point(aes(y = acc, color = "Acc")) + labs(color = "Type")

ggsave("history.png")
```

如果脚本成功运行的话，会在相同目录下产生一份名为`history.png`的图片，那么恭喜，安装成功了。
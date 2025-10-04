---
title: 深度学习入门-感知机和神经网络4-学习
date: 2025-10-03 16:07:25
categories:
- AI
tags:
- AI
- Deep Learning
- read
---

##  《深度学习入门：基于Python的理论与实现》 神经网络的学习

 [日]斋藤康毅

### 数值微分

#### 导数

10分钟内跑了2千米，每分钟跑了200米，虽然计算了1分钟的变化量200米，但这个是平均值。

导数表示某个瞬间的变化量，用公式表示：
$$
\frac{df(x)}{dx}=\lim_{h \to 0} \frac{f(x+h)-f(x)}{h}
$$
$\frac{df(x)}{dx}$表示f(x)关于x的导数，即f(x)相对于x的变化程度。x的“微小变化”(h无限趋近0)将导致函数f(x)的值在多大程度上发生变化。

##### 实现导数计算

```python
# 不好的实现示例
def numerical_diff(f, x):
    h = 10e-50
    return (f(x+h) - f(x)) / h
```

`numerical_diff(f, x)`的名称来源于数值微分的英文numerical differentiation。这个函数有两个参数，即`函数f`和`传给函数f的参数x`，这个实现有两个问题：

1. 10e-50（有50个连续的0的“0.00 ... 1”）这个微小值会因python中的舍入误差变为0
2. “真的导数”对应函数在x处的斜率（称为切线），但上述实现中计算的导数对应的是(x+h)和x之间的斜率。因此，真的导数（真的切线）和上述实现中得到的导数的值在严格意义上并不一致。这个差异的出现是因为h不可能无限接近0。

对于问题1，可以将h的值设置为1e-4;

对于问题2，我们可以计算函数f在(x+h)和(x-h)之间的差分，因为这种计算方法以x为中心，计算它左右两边的差分，所以也称为中心差分（而(x+h)和x之间的差分称为前向差分）。

```python
def numerical_diff(f, x):
    h = 1e-4 # 0.0001
    return (f(x+h) - f(x-h)) / (2*h)
```

利用微小的差分求导数的过程称为**数值微分(numerical differentiation)**。而基于数学式的推导求导数的过程，则用“解析性”(analytic)一词，称为**“解析性求解”或者“解析性求导”**。比如$y=x^2$的导数，可以通过$\frac{dy}{dx}=2x$解析性地求解出来。因此，当x= 2时，y的导数为4。解析性求导得到的导数是不含误差的“真的导数"

对函数$f(x) = 0.01x^2+0.1x$ 计算x为5的导数，使用数学分析的方案$\frac{dy}{dx}=0.02x+0.1$，当x=5时，得到微分值为0.2，和使用数值微分计算出来0.19999是近似相同的

```python
import numpy as np
import matplotlib.pylab as plt

def numerical_diff(f, x):
    h = 1e-4
    return (f(x+h) - f(x-h)) / (2*h)

def test_func(x):
    return 0.01*x**2 + 0.1*x

def tangent_line(f, x):
    d = numerical_diff(f, x)
    print(d) # 0.1999999999990898
    y = f(x) - d*x 
    return lambda t: d*t + y # 使用计算的出来的导数值绘制斜率

def plot_test_func():
    x = np.arange(0.0, 20.0, 0.1) # 以0.1为单位，从0到20的数组x
    y = test_func(x)
    tf = tangent_line(test_func, 5)
    y2 = tf(x)
    plt.xlabel("x")
    plt.ylabel("f(x)")
    plt.plot(x, y)
    plt.plot(x, y2)
    plt.show()
```

##### 偏导数

 普通导数处理的是单变量函数 ，对有多个变量的函数的求导数称为偏导数。

函数 $f(x_1, x_2, ..., x_n)$ 对其某个变量$x_i$的偏导数记为 $\frac{\partial f}{\partial x_i}$。它表示函数$f$保持其他变量不变时，相对于变量 $x_i$的变化率。公式为
$$
\frac{\partial f}{\partial x_i} = \lim_{h\to 0} \frac {f(x_1, x_2,..,x_i+h,..,x_n)-f(x_1, x_2,..,x_i,..,x_n)} {h}
$$
本质上和一个变量的函数导数相同，只是其他变量都是某一个固定值。

对于一个二元函数
$$
f(x_0, x_1) = x_0^2 + x_1^2
$$

它的图形如下是个三维曲面，最低点在(0, 0)，由于它有两个变量，所以有必要区分对哪个变量求导数，即对$x_0$和$x_1$两个变量中的哪一个求导数。

![2_var_fun_plot_3d](../../uploads/ai/2_var_fun_plot_3d.png)
![2_var_fun_plot_3d](/uploads/ai/2_var_fun_plot_3d.png)

当$x_1=4$时，函数变为$f(x_0) = x_0^2 + 4^2$，变成一个只有一个变量的函数，计算这个函数对$x_0$求导，当$x_0=3$时，导数值为6.00000000000378。

偏导数和单变量的导数一样，都是求某个地方的斜率。不过，**偏导数需要将多个变量中的某一个变量定为目标变量，并将其他变量固定为某个值**。

#### 梯度

梯度指示的方向是各点处的函数值变化最多的方向。

一起计算$x_0$和$x_1$的偏导数，例如$x_0=3, x_1=4$时，$(x_0, x_1)$的偏导数$\big(\frac{\partial f}{\partial x_0},\frac{\partial f}{\partial x_1}\big)$。这种由全部变量的偏导数汇总而成的向量称为**梯度(gradient)**。例如对输入x=[3,4] 计算上面函数的梯度，得到的向量为[6, 8]。

```python
def test_func_2(x):
   return np.sum(x**2) # 每个元素的平方和

def numerical_gradient(f, x):
    h = 1e-4 # 0.0001
    grad = np.zeros_like(x) # 生成和x形状相同的数组其中的值都为0
	# 分别对每一个元素计算导数，以idx = 0为例
    for idx in range(x.size):
        tmp_val = x[idx]        
        x[idx] = float(tmp_val) + h #x = [3.0001, 4]
        fxh1 = f(x) # f(x+h)的计算 # 3.0001**2+4**2 = 25.0006
        
        x[idx] = float(tmp_val) - h #x = [2.9999, 4]
        fxh2 = f(x) # f(x-h)的计算 # 2.9999**2 + 4**2 = 24.99940001  

        grad[idx] = (fxh1 - fxh2) / (2*h) # grad[0] = 5.99995
        x[idx] = tmp_val # 还原值

    return grad

if __name__ == '__main__':
    print(numerical_gradient(test_func_2, np.array([3.0, 4.0]))) #[6. 8.]
```

用图形表示元素值为负梯度的向量（导数值取负数），$f(x_0, x_1) = x_0^2 + x_1^2$的梯度呈现为有向向量（箭头）。梯度指向函数$f(x_0, x_1)$的“最低处”（最小值），就像指南针一样，所有的箭头都指向同一点。其次，我们发现离“最低处”(0, 0)越远，箭头越大。当$x_1=0$时，$f(x_0, x_1) = x_0^2$，是一个标准的一元二次函数，$x_0$的值越大，对应的导数越大，斜率值也越大，$x_0$变化一点后，y的变化也大。**对于梯度，更关心的是变化方向**，下图中的代码使用`-grad[0], -grad[1]`梯度的负值来绘图，所以是指向函数极小值。可以这样理解：对函数$f(x_0, x_1)$位于坐标(3, 4)时，它沿着梯度(6, 8)方向，变化最快。所以通过负梯度，就可以最快的找到函数的极小值。下图中，坐标为(2, -2)时，计算出的梯度值为(4, -4)，取反后的梯度值为(-4, 4)，所以从(2, -2)这个位置出发，向(2-4, -2+4)方向即x0-2，x1+2的方向，函数值向最小值方向变化最快，如图右下角的箭头向左上45度，就是它变小最快的方向。

![gradient_arrow](../../uploads/ai/gradient_arrow.png)
![gradient_arrow](/uploads/ai/gradient_arrow.png)

对应代码

```python
def test_func_2(x):
    if x.ndim == 1:
        return np.sum(x**2)
    else:
        return np.sum(x**2, axis=1)

def _numerical_gradient_no_batch(f, x):
    h = 1e-4 # 0.0001
    grad = np.zeros_like(x) # 生成和x形状相同的数组其中的值都为0

    for idx in range(x.size):
        tmp_val = x[idx]        
        x[idx] = float(tmp_val) + h
        fxh1 = f(x) # f(x+h)的计算
        
        x[idx] = float(tmp_val) - h
        fxh2 = f(x) # f(x-h)的计算

        grad[idx] = (fxh1 - fxh2) / (2*h)
        x[idx] = tmp_val # 还原值

    return grad

def numerical_gradient(f, X):
    if X.ndim == 1:
        return _numerical_gradient_no_batch(f, X)
    else:
        grad = np.zeros_like(X)  
        print(grad.shape) # (2, 324)
        for idx, x in enumerate(X): # idx 为行号索引 0-1
            print("shape of x:", x.shape) #shape of x: (324,)
            grad[idx] = _numerical_gradient_no_batch(f, x)
        
        return grad
    
if __name__ == '__main__':
    # 两行数据，每一行18个数据
    x0 = np.arange(-2, 2.5, 0.25)
    x1 = np.arange(-2, 2.5, 0.25)
    # [X,Y] = meshgrid(x,y) 基于向量 x 和 y 中包含的坐标返回二维网格坐标。X 是一个矩阵，每一行是 x 的一个副本；Y 也是一个矩阵，每一列是 y 的一个副本。坐标 X 和 Y 表示的网格有 length(y) 个行和 length(x) 个列。
    X, Y = np.meshgrid(x0, x1)
    print(X.shape) #(18, 18)
    X = X.flatten() #(324,)
    Y = Y.flatten()
    # np.array([X, Y])的shape 为(2, 324)
    grad = numerical_gradient(test_func_2, np.array([X, Y]) )
    
    plt.figure()
    # quiver([X, Y], U, V, [C], **kwargs) X, Y定义箭头位置，U, V定义箭头方向， C可选择设置颜色
    # angles="xy"：数据坐标中的箭头方向，即箭头从（x，y）指向（x+u，y+v）。使用它，例如绘制梯度场。
    # 这里相当于绘制(x0, x1)构成的每一个点的指向这个点对应的导数（-grad[0], -grad[1])表示箭头方向   
    plt.quiver(X, Y, -grad[0], -grad[1],  angles="xy",color="#666666")
    plt.xlim([-2, 2])
    plt.ylim([-2, 2])
    plt.xlabel('x0')
    plt.ylabel('x1')
    plt.grid()
    plt.draw()
    plt.show()
```

#### 梯度法

一般而言，损失函数很复杂，参数空间庞大，我们不知道它在何处能取得最小值。而通过巧妙地使用梯度来寻找函数最小值（或者尽可能小的值）的方法就是梯度法。

梯度表示的是各点处的函数值减小最多的方向，无法保证梯度所指的方向就是函数的最小值或者真正应该前进的方向。实际上，在复杂的函数中，梯度指示的方向基本上都不是函数值最小处。



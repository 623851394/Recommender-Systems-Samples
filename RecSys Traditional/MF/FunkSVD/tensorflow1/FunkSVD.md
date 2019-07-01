
@[toc]
## 写在前面
> 对奇异值分解数学原理有疑惑的，请阅读：《[传统推荐算法(一)利用SVD进行推荐（3）6个层面透彻了解奇异值分解》](https://blog.csdn.net/xxiaobaib/article/details/92012829)
> 参考代码：https://github.com/felipessalvatore/Recommender
> 重构代码：https://github.com/wyl6/Traditional-Classical-Conventional-Recommender-Systems/tree/master/MF/FunkSVD
> 参考代码中将FunkSVD和NSVD写在一起，我觉得作者写得还不错，看起来有scikit-learn代码的风格。就参考了他的代码中SVD那一部分练练手。

## 1.传统SVD分解的缺陷
在《传统推荐算法(一)（4）实战中，我们讲了利用SVD实现对物品的隐特征降维，这是SVD在推荐中的一种应用。实际上还有一种应用，就是利用SVD对评分矩阵进行分解，然后利用降维后的U,S,V对矩阵进行还原，原先矩阵中空缺的部分就可以得到填充值。第一种应用中SVD就是作为降维技术，优化相似度计算的过程；第二种利用这种矩阵分解技术分解后再填充，将填充的结果作为预测值。

第一种缺点已经分析过了，本文重点看一下第二种应用的缺点，这种应用就是FunkSVD改进前的SVD推荐。




### 1.1 稀疏矩阵的缺失值问题

[2]中指出“奇异值分解作为矩阵分解中最基本的方法，通常都是作用于完整的矩阵（即矩阵中不存在元素缺失的现象），即要求矩阵必须是稠密矩阵”。[3]中指出“这种方法存在一个致命的缺陷——奇异值分解要求矩阵是“稠密”的”。

注：这里的“稠密”指的是矩阵不能有缺失元素(有缺失元素分解个锤子)。和下文中的稠密有所区别。

由于实际中大多数评分矩阵都会有缺失值，所以用这种SVD分解填充的话，第一步都会想办法对缺失值做一个“粗糙的”填补。这个填补会造成问题。

### 1.2 稀疏矩阵的填补问题
> 历史上对缺失值的研究有很多，对于一个没有被打分的物品来说，到底是应该给它补一个 0 值，还是应该给它补一个平均值呢？由于在实际过程中，元素缺失值是非常多的，这就导致了早期的SVD不论通过以上哪种方法进行补全在实际的应用之中都是很难被接受的。

“粗糙的”填充会造成两个明显的问题[5]：第一，增加了数据量，增加了算法复杂度。第二，简单粗暴的数据填充很容易造成数据失真。我们发现很多时候都是直接补0，得到一个非常稀疏的打分矩阵。这种补零的情况还会造成空间的浪费。

### 1.3 SVD计算复杂度问题

[1]中指出：
> SVD 分解的复杂度比较高，假设对一个 𝑚×𝑛 的矩阵进行分解，时间复杂度为 𝑂(𝑛2∗𝑚+𝑛∗𝑚2)，其实就是 𝑂(𝑛3)。对于 m、n 比较小的情况，还是勉强可以接受的，但是在推荐场景的海量数据下，m 和 n 的值通常会比较大，可能是百万级别上的数据，这个时候如果再进行 SVD 分解需要的计算代价就是很大的。

[4]中指出：
> SVD计算代价很大，时间复杂度是3次方的，空间复杂度是2次方的。虽然有一些号称并行的SVD算法，但据我所知，如果不能共享内存，SVD很难并行化。

### 1.4 SVD分解对内存的消耗问题

对于1万个用户，1000万个物品的评分矩阵，我们简单估计下它需要的内存。假设每个评分用4个字节表示，那么SVD分解时，将这个完整的矩阵加载到内存中，大概需要372G内存。内存是否能容纳是个大问题。

### 1.5 SVD的特征表达问题
在在《传统推荐算法(一)（4）实战中，我们分析过这个，SVD分解方式比较单一，分解得到的特征表示是否一定是用户/物品的一种比较好的表示呢？从这个角度看，获取的特征的方式不够灵活，那么这样无论是在第一种降维的应用，还是在第二种填充的应用中，都不能保证获取好的结果。

所以，用SVD进行评分矩阵填充时，并不会采用正统的SVD分解，而是采用一种灵活的版本，请见1.6。

### 1.6稀疏矩阵的SVD分解问题

由于奇异值分解的计算过程较慢，所以在实际使用SVD分解进行填充预测时，使用的是“伪SVD”分解(图片来自孟一凡大佬的截图)：
![1753035041.jpg](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/1753035041.jpg)

这里的U和V是用户和物品的隐向量，Σ是对角阵，通过最小化平方误差的解析方法来求解。最小化平方误差通常用最小二乘法(ALS)或者梯度下降。由于正统的SVD分解出正交矩阵会比较慢，所以在实际操作中只取SVD精髓，训练出用户和物品的隐向量U和V即可，不要求U和V正交。

我们可以发现，U,Σ,V中有许多参数需要确定，如果矩阵过于稀疏，显然模型的效果不会理想。如果想要保证模型效果，就必须保证训练数据充分，要求矩阵不能稀疏，毕竟模型参数太多了。孟一凡大佬的话说:
> “如果数据很稀疏，模型能利用的信息是有限的，算法填充矩阵的能力也是有限的，所以稠密。”

那这就尴尬了，实际中的评分矩阵稠密了还需要填充吗，还需要预测吗？这种改进后的“伪SVD”依然不够给力。

## 2.隐语义模型LFM

### 2.1 大名鼎鼎的LFM
之前我们分析过，SVD分解后：

![](https://github.com/wyl6/wyl6.github.io/blob/master/imgs_for_blogs/recommender_traditional/MF/SVD/1.bmp?raw=true)

其实可以简化为：

![](https://github.com/wyl6/wyl6.github.io/blob/master/imgs_for_blogs/recommender_traditional/MF/SVD/3.bmp?raw=true)

在此基础上，我们从“伪SVD”中，可以观察到：
![图片来自[6]](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/20190623224258.png)

倘若将奇异值融合到u<sub>i</sub>和v<sub>i</sub>和中去，就可以将矩阵分解转化为一个参数估计问题[6]：
![20190623225630.png](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/20190623225630.png)

这样得到的用户和物品隐特征矩阵虽然不一定具备经典奇异值分解的正交矩阵性质，但是用来作为用户和物品的特征表达(隐因子)也足够了。
> 并且这隐含着机器学习的常用方法论：只要隐因子在估计已知评分时足够准，那么辅以一些正则化保证泛化能力之后它在未知评分上的预测也能满足精度要求[6]。
> 
我们可以发现，相比刚才的“伪SVD”，这个模型也可以获得用户和物品的特征表达，并且将原来的奇异值融合进了特征中，特征包含了更多的信息；同时参数数量大大下降，相同的训练数据，这个模型显然可以拥有更好的性能。**这个模型就是大名鼎鼎的LFM模型。**


**2006年，FunkSVDSimon Funk 在博客上公布了一个算法(Funk-SVD)，这个算法后来被Netflix Prize 的冠军 Koren 称为 Latent Factor Model(LFM)。[7]**

LFM通过以下公式来计算用户u对物品i的评分：

![Screenshot from 2019-06-22 11-10-50.png](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/Screenshot%20from%202019-06-22%2011-10-50.png)

我们可以把p<sub>uk</sub>看成是用户u对第k个隐类的兴趣，把q<sub>ik</sub>看成是物品i和第k个隐类的相关性。

### 2.2 损失函数及训练
我们借助[7]中内容简要看一下。SVD损失函数为：

![20190624094234.png](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/20190624094234.png)

可以通过交替最小二乘法或者梯度下降法来训练模型。比如用梯度下降法，那么损失函数偏导为：
![20190624094746.png](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/20190624094746.png)

参数沿最速下降方向前进时，递推公式为：
![20190624094846.png](https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/20190624094846.png)

## 3.FunkSVD及其变种
FunkSVD的变种很多，简单整理一下，供大家参考。Matrix Factorization(MF)可以看成是LFM带起来的一种解决推荐问题的思路，有的人直接叫LFM为MF模型。但是现在矩阵分解技术很多，不建议这样叫，不然别人可能不知道你说的是矩阵分解这一类技术还是LFM这个模型。

| 算法 | 别名 | 内容 |
| :-----| :----: | :----: |
| SVD | traditional SVD | 奇异值分解 |
| FunkSVD | LFM, basic MF, MF | LFM |
| bias SVD | bias MF | LFM+偏置项 |
| regularized SVD | regularized MF | LFM+正则项 |
| PMF | * | LFM+正则项的概率版本 |
| SVD++ | * | LFM+正则项+隐性反馈 |
| NMF | * | 对隐向量非负限制，可用在bias SVD等不同模型上 |
| NSVD | * | 学习两个物品隐因子，而非学习用户、物品隐因子 |
| WRMF | * | 引入时间信息，可用在bias SVD等不同模型上 |





## 3.Tensorflow实现FunkSVD
FunkSVD可以说是一个上古战神了，模型比较简单，网上有不少用Python写的，代码都不长。我们看看tensorflow如何实现，首先看一下各变量的定义：
```Python
import numpy as np
import tensorflow as tf
import time
import os
from util import rmse, status_printer

def inference_svd(batch_user, batch_item, num_user, num_item, dim=5):
    
    with tf.name_scope('Declaring_variables'):
        
        ## obtain global bias, user bias, item bias
        w_bias_user = tf.get_variable('num_bias_user', shape=[num_user])
        w_bias_item = tf.get_variable('num_bias_item', shape=[num_item])
        bias_global = tf.get_variable('bias_global', shape=[])
        bias_user = tf.nn.embedding_lookup(w_bias_user, 
                                           batch_user, 
                                           name='batch_bias_user')
        bias_item = tf.nn.embedding_lookup(w_bias_item, 
                                           batch_item, 
                                           name='batch_bias_imte')
        
        ## obtain user embedding, item embedding
        initializer = tf.truncated_normal_initializer(stddev=0.02)
        w_user = tf.get_variable('num_embed_user', 
                                 shape=[num_user, dim], 
                                 initializer=initializer)
        w_item = tf.get_variable('num_embed_item', 
                                 shape=[num_item, dim], 
                                 initializer=initializer)
        embed_user = tf.nn.embedding_lookup(w_user, 
                                            batch_user, 
                                            name='batch_embed_user')
        embed_item = tf.nn.embedding_lookup(w_item, 
                                            batch_item, 
                                            name='batch_embed_item')
```
实际上这份代码可以做更多扩展，不仅仅加入偏置项和正则项。然后我们看一下预测值：
```Python
        ## obtain r_ij = p_i*q_j
        infer = tf.reduce_sum(tf.multiply(embed_user, embed_item), 1) 
        
        ## obtain bias
        infer = tf.add(infer, bias_global)
        infer = tf.add(infer, bias_user)
        infer = tf.add(infer, bias_item, name='svd_inference')
```
然后看看正则项定义：
```Python
## obtain regularizer
        l2_user = tf.sqrt(tf.nn.l2_loss(embed_user))
        l2_item = tf.sqrt(tf.nn.l2_loss(embed_item))
        l2_sum = tf.add(l2_user, l2_item)
        bias_user_sq = tf.square(bias_user)
        bias_item_sq = tf.square(bias_item)
        bias_sum = tf.add(bias_user_sq, bias_item_sq)
        regularizer = tf.add(l2_sum, bias_sum, name='svd_regularizer')
        
        regularizer = tf.add(tf.nn.l2_loss(embed_user), 
                             tf.nn.l2_loss(embed_item), 
                             name="svd_regularizer")
```
那么损失函数就很清晰了：
```Python
def loss_function(infer, regularizer, batch_rate, reg):
    '''
    calculate loss = loss_l2+loss_regularizer
    '''
    loss_l2 = tf.square(tf.subtract(infer, batch_rate))
    reg = tf.constant(reg, dtype=tf.float32, name='reg')
    loss_regularizer = tf.multiply(regularizer, reg)
    loss = tf.add(loss_l2, loss_regularizer)
    return loss
```
训练过程使用:
```Python
self.train_op = optimizer.minimize(self.loss, global_step=global_step)
```
就不多说了。这份代码是我从一份代码里重构出来的，推荐大家阅读一下。不推荐跟着手写一遍，有点长，想手写的建议找个python版本的，简单好用。

等写SVD++那一部分的时候，尽量找个短的，毕竟这些代码只是帮助理解，真用的时候直接调成熟的包更好些。


## 参考
- [1] https://lumingdong.cn/recommendation-algorithm-based-on-matrix-decomposition.html#ref-footnote-4-1
- [2] https://zhuanlan.zhihu.com/p/25512080
- [3] https://zhuanlan.zhihu.com/p/34497989
- [4] https://www.zhihu.com/question/22572629
- [5] http://baogege.info/2014/10/19/matrix-factorization-in-recommender-systems/
- [6] https://zhuanlan.zhihu.com/p/38630064
- [7] 项亮. 推荐系统实践[J]. 北 京: 人 民 邮 电 出 版 社, 2012: 39-44.

## 公众号
更多精彩内容请移步公众号:推荐算法工程师

<img src="https://raw.githubusercontent.com/wyl6/wyl6.github.io/master/imgs_for_blogs/deep_learning/dnn/tensorflow_mnist/wechat.jpg" width = "200" height = "200" />

感觉公众号内容不错点个关注呗
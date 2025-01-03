# 模型量化

量化将神经网络的浮点算法转换为定点，修改网络中每个参数占用的比特数，从而减少模型参数占用的空间。

## 量化基本方法

### k-means 量化

**压缩**：K-means将权重值聚类成K类，每个类通过索引来表示。权重值不再直接存储，而是存储一个表示该权重所属簇的索引（例如，0代表第一类权重，1代表第二类权重，以此类推）。

**推理**：在推理时，我们读取**转换表**，根据存储的**索引值**找到对应的质心值，然后替代原始权重进行计算。

**训练**：在训练时，我们将权重按聚类方式进行相加，反向传播到转换表，更新转换表的值。

### 线性量化

线性量化是将浮点数（实数）转换为定点整数的过程，通常用于神经网络的权重和激活值的压缩。在这个过程中，浮点数和定点数之间的转换通过一个线性变换公式来实现。

## 训练后量化 (Post-Training Quantization)

训练后量化（PTQ）是指在模型训练完成后对模型进行量化处理，通常称为离线量化。根据量化时零点是否为零，PTQ 可分为对称量化和非对称量化。根据量化的粒度，PTQ 又可以分为逐张量量化、逐通道量化和组量化。

量化过程中会出现精度损失，因此如何选择合适的量化参数（如缩放因子、零点等）以尽量减少对准确性的影响是一个关键问题。量化误差主要来源于两方面：**clip操作**和**round操作**。因此，如何合理计算动态量化参数，以及理解round操作的影响，成为需要关注的重点。

### 量化粒度

量化的粒度直接影响模型精度，选择合适的粒度可以在减少存储需求的同时，尽量保持模型的精度。

- **逐张量量化（Per-Tensor Quantization）**：对每一层的整个张量进行统一量化，在张量内的所有元素使用相同的量化参数。然而，张量内不同元素的数值范围可能不同，这会导致精度下降。例如，如果一个通道的数据范围较大，而另一个较小，共享相同的量化参数会导致较大的量化误差。
- **逐通道量化（Channel-wise Quantization）**：按通道进行量化，即每个通道的数据使用不同的量化参数。这种方法能够较好地适应每个通道的数值范围，减少量化误差，因此通常适用于卷积神经网络（CNN）等模型，因为卷积层的不同通道具有不同的权重范围。尽管逐通道量化能减少误差，但会增加存储需求，因为每个通道都需要存储独立的缩放因子和零点。
- **组量化（Group Quantization）**：将数据拆分成多个小组，每组共享一个量化参数。例如，在每个通道内部，可以将通道的元素划分为若干个小组，对每个小组应用不同的量化因子。这种方法尝试平衡精度和存储效率，适合较大规模的模型。

## 量化感知训练 (Quantization-Aware Training, QAT)

量化感知训练（QAT）是在模型训练过程中模拟量化操作，目的是使模型能够适应量化后带来的精度损失，从而确保量化后的模型与原始模型的表现尽可能接近。QAT 通过在训练时引入模拟量化算子来模拟推理阶段的舍入和裁剪操作，将量化误差融入到训练过程中，从而使得模型在量化后能够保持较高的性能。

### 前向传播

在量化感知训练中，前向传播的过程如下：

- 上一层的输出传到下一层
- 本层权重经过量化反量化后得到输出
- 输出反量化后传到下一层

整个量化过程中算子的计算都是在高精度下完成的

### 反向传播

在反向传播中，量化后的权重是离散的，因此，计算梯度时需要处理量化操作带来的不可导问题。通常，量化后的权重不可导，因此需要通过“直通估计器”（Straight-Through Estimator, STE）来解决梯度计算问题，进行反向传播。

## 模型量化对象

**权重量化**是最常见的量化方法，旨在减少模型大小和内存占用

**激活量化**有助于减少内存使用并加速推理

**KV缓存量化**对于提高长序列生成任务的吞吐量至关重要

**梯度量化**主要用于减少分布式计算中的通信开销和训练过程中的成本

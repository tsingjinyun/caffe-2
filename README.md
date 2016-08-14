# Sparse deep neural networks with structured sparsity

This is a fork of [Caffe](http://caffe.berkeleyvision.org/) targeting on sparse convolutional neural networks with *structured sparisty* to speedup DNN evaluation in computation- and memory-limited devices.

Technical details are in our NIPS 2016 paper.
Our *SSL (Structured Sparsity Learning)* method can dynamically learn a compact structure (less filters, less channels, smaller filter shapes and less layers) of deep neural networks, achieving speedups of AlexNet by 3.1X in GPUs and 5.1X in CPUs, using off-the-shelf GEMM in BLAS (e.g. MKL in CPUs and cuBLAS in nvidia GPUs). Alternatively, a variant of our method can improve accuracy of AlexNet by ~1%. Moreover, our results can also reduce the number of layers in Deep Residual Networks (ResNets) meanwhile improving its accuracy.

## Motivation
Deep neural networks can be very sparse (>90%), after optimization by L1 regularization or connection pruning. Although the model size can be compressed, the computation cannot be sped up because of the poor cache locality and jumping memory access pattern.

![Alt text](/models/bvlc_reference_caffenet/speedups.png?raw=true "Speedup vs. sparsity")

The above figure shows the speedups are very limited (sometimes even slows down), althrough the sparsity can be as high as >90%. Our *SSL* method can train DNNs with structured sparsity, which results in very good locality and memory access pattern.

## Caffemodel
Uploaded in [Caffe Model Zoo](https://github.com/BVLC/caffe/wiki/Model-Zoo#learning-structured-sparsity-in-deep-neural-networks).

## HowTo and Features
### Train sparsity-structured convolutional neural networks 
You can use our trained caffemodel in the model zoo, or train it by yourselves.

1. Stabilizing sparsity
  - weights smaller than a threshold ([0.0001](http://www.cv-foundation.org/openaccess/content_cvpr_2015/papers/Liu_Sparse_Convolutional_Neural_2015_CVPR_paper.pdf)) are zeroed out after updating weights
2. New [caffe.proto](https://github.com/wenwei202/caffe/blob/scnn/src/caffe/proto/caffe.proto) configurations
  - `block_group_decay` in `SolverParameter`: weight decay by group lasso regularization on each group(block) in weight matrix, block are configured by `block_group_lasso` in each `ParamSpec` (e.g. weights);
  - `connectivity_mode` in `LayerParameter` can permanently prune zero-weighted connections;
  - local `regularization_type` ("L1/L2") is supported for each `ParamSpec` (e.g. weights)
3. In CPU mode, use Intel mkl blas.

### Test/Evaluate sparsity-structured convolutional neural networks 
  - [conv_mode](https://github.com/wenwei202/caffe/blob/scnn/src/caffe/proto/caffe.proto#L637) in `ConvolutionParameter` configures the computation modality of convolution (GEMM, CSR, Concatenation, etc.). Following is an example to configure `deploy.prototxt` so that the matrix multiplication is operated by sparse weight matrix * dense feature map matrix.
```
layer {
  name: "conv2"
  type: "Convolution"
  bottom: "norm1"
  top: "conv2"
  convolution_param {
    num_output: 256
    pad: 2
    kernel_size: 5
    group: 2
    conv_mode: LOWERED_CSRMM # sparse weight matrix in CSR format * lowered feature maps
    # conv_mode: LOWERED_GEMM # default original matrix multiplication 
  }
}
```


Please cite our NIPS 2016 paper and Caffe if it helps you:

    @incollection{Wen_NIPS2016,
    Title = {Learning Structured Sparsity in Deep Neural Networks},
    Author = {Wen, Wei and Wu, Chunpeng and Wang, Yandan and Chen, Yiran and Li, Hai},
    bookTitle = {Advances in Neural Information Processing Systems},
    Year = {2016}
    }
    
    @article{jia2014caffe,
      Author = {Jia, Yangqing and Shelhamer, Evan and Donahue, Jeff and Karayev, Sergey and Long, Jonathan and Girshick, Ross and Guadarrama, Sergio and Darrell, Trevor},
      Journal = {arXiv preprint arXiv:1408.5093},
      Title = {Caffe: Convolutional Architecture for Fast Feature Embedding},
      Year = {2014}
    }

### Notes
Speed is compared by matrix-matrix multiplication (GEMM) in each convolutional layer, by a layer-by-layer fashion. The speedup of the total time may be different, because
  1. The implementation of lowering convolution to GEMM is not efficient in Caffe, especially in CPU mode.
  2. After the time of GEMM is squeezed, the computation time of other layers (e.g. pooling layers) comes to the surface.

However, the lowering and pooling can also be optimized by programming tricks. Please refer to our paper of [Holistic SparseCNN](https://arxiv.org/abs/1608.01409) and [intel branch](https://github.com/wenwei202/caffe/tree/intel):

    @article{jia2014caffe,
      Author = {Park, Jongsoo and Li, R. Sheng and Wen, Wei and Li, Hai and Chen, Yiran and Dubey, Pradeep},
      Journal = {arXiv preprint arXiv:1608.01409},
      Title = {Holistic SparseCNN: Forging the Trident of Accuracy, Speed, and Size},
      Year = {2016}
    }

  

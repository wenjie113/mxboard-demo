# MXBoard Demo
Understanding the underlying mechanism of complex neural networks can be difficult.
The invention of
[TensorBoard](https://www.tensorflow.org/programmers_guide/summaries_and_tensorboard)
makes common operations of running neural networks visible. It greatly facilitates
people understanding, debugging, and optimizing deep learning programs.
In order for MXNet users to use TensorBoard, we developed a tool called
[MXBoard](https://github.com/awslabs/mxboard) for logging MXNet data
in the format recognizable to TensorBoard.
In this demo, we are going to demonstrate the usage of MXBoard
for MXNet users to train and understand neural networks in a more intuitive way.


## Monitoring Training MNIST Model
Let's borrow the script of training the MNIST model from the
[Gluon example](https://github.com/apache/incubator-mxnet/blob/master/example/gluon/mnist.py)
to monitor the training progress in TensorBoard. You can find the code in `train_mnist.py`.
Note that here we define the network using `HybridSequential`,
instead of `Sequential` as in the Gluon example. This is because that we want to plot
the model graph in TensorBoard and MXBoard only accepts `HybridBlock`s for models built
using Gluon interfaces.

We would like to highlight the snippets in the `train()` function where MXBoard comes into play.
1. Define a `SummaryWriter` object for writing MXNet data to event files under the `./logs`
directory and flushing writes every five seconds in order to view them in TensorBoard
in training.
```python
sw = SummaryWriter(logdir='./logs', flush_secs=5)
```

2.  For the first batch of images of every epoch, display them in TensorBoard sequentially.
We want to verify that these images are different since we set
`shuffle=True` in the `DataLoader` for the training dataset.
```python
if i == 0:  # i here represents the minibatch id
    sw.add_image('minist_first_minibatch', data.reshape((opt.batch_size, 1, 28, 28)), epoch)
```

![mnist_image_samples](https://github.com/reminisce/mxboard-demo/blob/master/pic/mnist_image_samples.png)

3. Plot the graph of the MNIST model.
```python
if epoch == 0:
    sw.add_graph(net)
```

![mnist_graph](https://github.com/reminisce/mxboard-demo/blob/master/pic/mnist_graph.png)

4. Plot the histograms of the gradients of parameters for every epoch.
```python
grads = [i.grad() for i in net.collect_params().values()]
assert len(grads) == len(param_names)
# logging the gradients of parameters for checking convergence
for i, name in enumerate(param_names):
    sw.add_histogram(tag=name, values=grads[i], global_step=epoch, bins=num_bins)
```

![mnist_params_histograms](https://github.com/reminisce/mxboard-demo/blob/master/pic/mnist_params_histograms.png)


5. Plot training and validation accuracy curves.
```python
name, acc = metric.get()
print('[Epoch %d] Training: %s=%f' % (epoch, name, acc))
# logging training accuracy
sw.add_scalar(tag='train_acc', value=acc, global_step=epoch)

name, val_acc = test(ctx)
# logging the validation accuracy
print('[Epoch %d] Validation: %s=%f' % (epoch, name, val_acc))
sw.add_scalar(tag='valid_acc', value=val_acc, global_step=epoch)
```

![train_valid_accuracy_curves](https://github.com/reminisce/mxboard-demo/blob/master/pic/train_valid_accuracy_curves.png)


6. Close the `SummaryWriter`.
```python
sw.close()
```

## Visualizing Filters of ConvNets
This section visualizes the filters and outputs of the first convolution layers in three ConvNets:
[Inception-BN](http://data.mxnet.io/models/imagenet/inception-bn/),
[Resnet-152](http://data.mxnet.io/models/imagenet/resnet/152-layers/),
and
[VGG16](http://data.mxnet.io/models/imagenet/vgg/).

All three models are trained on [ImageNet](http://image-net.org/) dataset.
The outputs are generated by applying filters to an image selected from the
[validation dataset](http://data.mxnet.io/data/val_256_q90.rec).
The three pictures in each example are original picture from the validation dataset, the filters of the first
convolution layer, and the outputs of the convolution layer after applying the filters to the original image.
Each filter in the middle picture corresponds to the output image of the same coordinate in the picture on its
right side.
The smooth and nicely-formed patterns in the filters indicate that the models have been well trained.
The gray-scale filters tend to extract the outlines of the object, while the colorful filters
focus on local features of the objects.
Run command
```bash
$ python plot_filter_and_output.py
```
to write filters and outputs to event files for visualization in TensorBoard.

- Inception-BN
![inception_bn_conv_1_weight_output](https://github.com/reminisce/mxboard-demo/blob/master/pic/inception_bn_conv_1_weight_output.png)

- Resnet-152
![resnet_152_conv0_weight_output](https://github.com/reminisce/mxboard-demo/blob/master/pic/resnet_152_conv0_weight_output.png)

- VGG16
![vgg16_conv1_1_weight_output](https://github.com/reminisce/mxboard-demo/blob/master/pic/vgg16_conv1_1_weight_output.png)


## Visualizing ConvNet Codes as Embeddings
Embeddings are the mathematical vector representation of real-world objects in high-dimensional space.
One can quantify the similarity between two objects by calculating the normalized
inner-product of their embedding vectors.
Furthermore, dimension-reduction algorithms, such as
[PCA](https://en.wikipedia.org/wiki/Principal_component_analysis) and
[t-SNE](https://lvdmaaten.github.io/tsne/), can collapse high-dimensional embedding vectors
into 2D or 3D space so that their spatial position can be visualized.

While human eyes recognize images by telling the difference of shapes, colors, environments etc. from one to another,
ConvNets classify images via special codes generated by the last fully-connected layer just before the classifier
such as softmax. These codes can be taken as the embeddings of the objects and we can plot them in TensorBoard.

Here we randomly selected 2,304 images from the [validation dataset](http://data.mxnet.io/data/val_256_q90.rec)
of ImageNet as follows.

![images_for_convnet_codes](https://github.com/reminisce/mxboard-demo/blob/master/pic/images_for_convnet_codes.png)

Then we collected their embeddings generated by the pre-trained model
[Resnet-152](http://data.mxnet.io/models/imagenet/resnet/152-layers/) without the softmax layer,
and wrote the embeddings to event files for visualization in TensorBoard.
Let's visualize the embeddings using t-SNE since it is known for the excellent capability
of preserving local structures of object relationship.
One can apply other pre-trained models to achieve the similar results as below.

![imagenet_resnet_152_embedding](https://github.com/reminisce/mxboard-demo/blob/master/pic/imagenet_resnet_152_embedding.png)

On the top-right corner of the TensorBoard GUI, you can enter the object type to search for the matched
objects in the canvas, and then zoom in to check whether similar objects are clustered. A well trained
ConvNet model should produce the codes that enable t-SNE to place similar objects in the same neighborhood,
and preserve the relative distances of objects in the high-dimensional space. Here we want to search for
images of dogs, and after zooming in, we observed clustering of dog images as follows.
To reproduce the result, type
```bash
$ python get_convnet_codes.py
```
to generate image embeddings. Note that if you don't have an Nvidia GPU on your machine,
please use the following command instead:
```base
$ python get_convnet_codes.py --ctx cpu
```
Then type
```bash
$ python plot_convnet_embedding.py
```
to write embeddings into event files for visualization in TensorBoard.

![imagenet_resnet_152_dog_cluster](https://github.com/reminisce/mxboard-demo/blob/master/pic/imagenet_resnet_152_dog_cluster.png)


# References
- https://github.com/awslabs/mxboard
- https://github.com/apache/incubator-mxnet
- [Understanding and Visualizing Convolutional Neural Networks](http://cs231n.github.io/understanding-cnn/)
- [TensorBoard: Embedding Visualization](https://www.tensorflow.org/versions/r1.1/get_started/embedding_viz)
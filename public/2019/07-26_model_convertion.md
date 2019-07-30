# ___2019 - 07 - 26 Model Convertion___

# MMDNN
## 安装
  - [microsoft/MMdnn](https://github.com/microsoft/MMdnn)
  - **安装**
    ```sh
    pip install mmdnn
    pip install -U git+https://github.com/Microsoft/MMdnn.git@master
    ```
## 基本命令
  - **模型可视化** [MMDNN Visualizer](http://mmdnn.eastasia.cloudapp.azure.com:8080/)
    ```sh
    mmdownload -f keras -n inception_v3
    mmtoir -f keras -w imagenet_inception_v3.h5 -o keras_inception_v3
    ```
    选择文件 keras_inception_v3.json
  - **mmdownload** 下载预训练好的模型
    ```py
    # 返回框架支持的模型
    mmdownload -f tensorflow

    # 下载指定的模型
    mmdownload -f tensorflow -n inception_v3
    ```
  - **mmvismeta** 可以使用 tensorboard 作为后端将计算图可视化
    ```py
    mmvismeta imagenet_inception_v3.ckpt.meta ./log
    ```
  - **mmtoir** 将模型转化为中间表达形式 IR (intermediate representation)，结果中的 json 文件用于可视化，proto / pb 用于描述网络结构模型，npy 用于保存网络数值参数
    ```py
    mmtoir -f tensorflow -n imagenet_inception_v3.ckpt.meta  -w inception_v3.ckpt --dstNode MMdnn_Output -o converted
    ```
  - **mmtocode** 将 IR 文件转化为指定框架下构造网络的原始代码，以及构建网络过程中用于设置权重的参数，结果生成一个 py 文件，与 npy 文件一起用于模型迁移学习或模型推断
    ```py
    mmtocode -f pytorch -n converted.pb -w converted.npy -d converted_pytorch.py -dw converted_pytorch.npy
    ```
  - **mmtomodel** 生成模型，可以直接使用对应的框架加载
    ```py
    mmtomodel -f pytorch -in converted_pytorch.py -iw converted_pytorch.npy -o converted_pytorch.pth
    ```
  - **mmconvert** 用于一次性转化模型，是 mmtoir / mmtocode / mmtomodel 三者的集成
    ```py
    mmconvert -sf tensorflow -in imagenet_inception_v3.ckpt.meta -iw inception_v3.ckpt --dstNode MMdnn_Output -df pytorch -om tf_to_pytorch_inception_v3.pth
    ```
  - **caffe alexnet -> tf tested**
    ```sh
    master branch with following scripts:
    $ python -m mmdnn.conversion._script.convertToIR -f caffe -d kit_imagenet -n examples/caffe/models/bvlc_alexnet.prototxt -w examples/caffe/models/bvlc_alexnet.caffemodel
    $ python -m mmdnn.conversion._script.IRToCode -f tensorflow --IRModelPath kit_imagenet.pb --dstModelPath kit_imagenet.py -w kit_imagenet.npy
    $ python -m mmdnn.conversion.examples.tensorflow.imagenet_test -n kit_imagenet.py -w kit_imagenet.npy --dump ./caffe_alexnet.ckpt
    Tensorflow file is saved as [./caffe_alexnet.ckpt], generated by [kit_imagenet.py] and [kit_imagenet.npy].
    ```
## 加载模型用于迁移学习
  - IR 文件转化为 tensorflow 模型
    ```sh
    # IR 文件转化为模型代码
    mmtocode -f tensorflow --IRModelPath converted.pb --IRWeightPath converted.npy --dstModelPath tf_inception_v3.py

    # 修改 is_train 为 True
    sed -i 's/is_train = False/is_train = True/' tf_inception_v3.py

    # IR 文件转化为 tensorflow 模型
    mmtomodel -f tensorflow -in tf_inception_v3.py -iw converted.npy -o tf_inception_v3 --dump_tag TRAINING
    ```
  - 模型加载
    ```py
    export_dir = "./tf_inception_v3"
    with tf.Session(graph=tf.Graph()) as sess:
        tf.saved_model.loader.load(sess, [tf.saved_model.tag_constants.TRAINING], export_dir)

        x = sess.graph.get_tensor_by_name('input:0')
        y = sess.graph.get_tensor_by_name('xxxxxx:0')
        ......
        _y = sess.run(y, feed_dict={x: _x})
    ```
## MMdnn IR 层的表示方式
  - [IR 层的 proto 说明文件](https://github.com/Microsoft/MMdnn/blob/master/mmdnn/conversion/common/IR/graph.proto)
  - IR 文件包含以下几个部分
    - **GraphDef** NodeDef / version
    - **NodeDef** name / op / input / attr
    - **Attrvalue** list / type / shape / tensor
    - **Tensorshape**  dim
    - **LiteralTensor** type / tensor_shape / values
  - **Graph** 描述网络模型，内部包含若干 node，对应网络模型的层，node 中的 input 描述每一层之间的输入输出连接关系
  - **NodeDef**
    - **name** 本层 name，在 Graph 中唯一
    - **op** 本层算子，算子描述见链接，在算子描述文件中，算子 name 唯一
    - **input** 用于描述本层输入关系，各层之间的输入输出关系靠此成员描述
    - **attr** attr 成员可以是 listvalue / type / shape / tensor
      - **list** 存储 list 成员，如 list(int) / list(float) / list(shape) 等，**data** 保存数值数据，类型可以是 bytes / int64 / float / bool
      - **type** 描述数值类型
      - **shape** 描述 tensor 形状
      - **tensor** 各 node 之间传递的 tensor
  - **TensorShape** 描述 tensor 的维度信息
  - **LiteralTensor** 存储 tensor, 在各 node 之间传递
***

# ONNX
  - [onnx/onnx](https://github.com/onnx/onnx)
  - [onnx/models](https://github.com/onnx/models)
  - [onnx/tutorials](https://github.com/onnx/tutorials)
## mxnet-model-server and ArcFace-ResNet100 (from ONNX model zoo)
  - [ArcFace-ResNet100 (from ONNX model zoo)](https://github.com/awslabs/mxnet-model-server/blob/master/docs/model_zoo.md/#arcface-resnet100_onnx)
  - [awslabs/mxnet-model-server](ttps://github.com/awslabs/mxnet-model-server)
  ```sh
  pip install mxnet-model-server
  mxnet-model-server --start --models arcface=https://s3.amazonaws.com/model-server/model_archive_1.0/onnx-arcface-resnet100.mar

  curl -O https://s3.amazonaws.com/model-server/inputs/arcface-input1.jpg

  curl -O https://s3.amazonaws.com/model-server/inputs/arcface-input2.jpg

  curl -X POST http://127.0.0.1:8080/predictions/arcface -F "img1=@arcface-input1.jpg" -F "img2=@arcface-input2.jpg"

  mxnet-model-server --stop
  ```
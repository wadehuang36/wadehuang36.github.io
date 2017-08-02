---
layout: post
title: Offline Image Classifier on Android
description: An example of using Tensorflow Mobile and MobileNet for image classifier offline.
tags: 
    - tensorflow
    - image classification
    - machine learning
---

# Offline Image Classifier on Android
One of the advantage of Tensorflow is that it has libraries for [Mobile](https://www.tensorflow.org/mobile/) devices such as iOS and Android. Tensorflow Mobile is not full functional as Desktop version. It cannot do training or building graph, but it can load trained models and run them. That meaning you can do Machine Learning methods on your phones, for example, image classification. This blog post will show you that how to apply trained model on Android.

[Example Source Code](https://github.com/wadehuang36/tensorflow_mobilenet_android_example/)

## Requirement
- [Tensorflow](https://www.tensorflow.org/install/)
- Python

> you can see my another post [TensorFlow Get Start in Python](https://wadehuang36.github.io/2017/07/07/TensorFlow-Get-Start.html) to know how to run TensorFlow in Docker.

## Step 1. Build and Train the Model
The most complicated thing in Machine Learning is building and training the models. it consumes a lot of time to build the models and train them. Therefore, I used a  trained model provided by Tensorflow which is [MobileNet](https://github.com/tensorflow/models/blob/master/slim/nets/mobilenet_v1.md) for this step. Download a checkpoint tarball and untar it for next step.

> There are other trained model provided by Tensorflow, see the [list](https://github.com/tensorflow/models/).

## Step 2. Freeze Your Model
The files you downloaded from step 1 are checkpoint files(data, index and meta) which just the states of variables in the graph. The model can be trained continually with these files, but since Tensorflow Mobile doesn't support training, it also doesn't support loading checkpoint files. Thus, we have to make it become protobuf binary with the graph and data together. This step Tensorflow calls `Freeze`. There is not one tool to do this job. On the Tensorflow git repositories, there are two Python or C++ scripts to do this.

- https://github.com/tensorflow/models/blob/master/slim/export_inference_graph.py
- https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/tools/freeze_graph.py

``` bash
# it is better to clone the repos, because these scripts have some dependencies.

python export_inference_graph.py --model_name=mobilenet_v1 \
    --default_image_size=224 \
    --output_file=mobilenet_v1.pb \

python freeze_graph.py --input_graph=mobilenet_v1.pb \
         --input_checkpoint=./checkpoints/mobilenet_v1.ckpt \
         --input_binary=true \
         --output_graph=mobilenet_v1.pb \
         --output_node_names=MobilenetV1/Predictions/Reshape_1
```

However, I combined the two scripts into one. There is one dependency, you have put this [script](https://github.com/tensorflow/models/blob/master/slim/nets/mobilenet_v1.py) on the same folder.


``` python
import tensorflow as tf

slim = tf.contrib.slim

# mobilenet_v1.py has to be put in the same folder
import mobilenet_v1 as network

arg_scope = network.mobilenet_v1_arg_scope()
network_fn = network.mobilenet_v1

image_size = 224
num_classes = 1001
checkpoints = "./checkpoints/mobilenet_v1.ckpt"
output = "./mobilenet_v1.pb"

# The name of the last step which is defined in mobilenet_v1.py
output_node_names = "MobilenetV1/Predictions/Reshape_1"

if not tf.train.checkpoint_exists(checkpoints):
    print("Input checkpoint '" + checkpoints + "' doesn't exist!")

tf.logging.set_verbosity(tf.logging.INFO)
with tf.Graph().as_default() as graph:

    placeholder = tf.placeholder(name='input', dtype=tf.float32,
                                 shape=[1, image_size, image_size, 3])

    with slim.arg_scope(arg_scope):
        network_fn(placeholder, num_classes, is_training=False)
        
    with tf.Session() as sess:
        var_list = {}
        reader = tf.train.NewCheckpointReader(checkpoints)
        var_to_shape_map = reader.get_variable_to_shape_map()
        for key in var_to_shape_map:
            try:
                tensor = sess.graph.get_tensor_by_name(key + ":0")
            except KeyError:
                # This tensor doesn't exist in the graph (for example it's
                # 'global_step' or a similar housekeeping element) so skip it.
                continue
            var_list[key] = tensor
            
        saver = tf.train.Saver(var_list=var_list)
        saver.restore(sess, checkpoints)
        
        output_graph_def = tf.graph_util.convert_variables_to_constants(
            sess,
            graph.as_graph_def(),
            output_node_names.split(","),
            variable_names_blacklist=None)

        with tf.gfile.GFile(output, 'wb') as f:
            f.write(output_graph_def.SerializeToString())
``` 

> See the [example](https://github.com/wadehuang36/notebooks/blob/master/machine-learning/tensorflow/Freeze%20Graph.ipynb) in Jupyter Notebook

### Labels
You can use the [labels](https://github.com/wadehuang36/tensorflow_mobilenet_android_example/blob/master/app/src/main/assets/labels.txt) I reformat or download [the origin file](https://raw.githubusercontent.com/tensorflow/models/master/inception/inception/data/imagenet_metadata.txt) from ImageNet.

### Test the protobuf binary is generated correctly
``` bash
# From https://github.com/tensorflow/tensorflow/blob/master/tensorflow/examples/label_image/label_image.py

python label_image.py --image=image_path \
    --graph=./mobilenet_v1.pb \
    --labels=labels.txt \ 
    --output_layer=MobilenetV1/Predictions/Reshape_1
```

> See the [example](https://github.com/wadehuang36/notebooks/blob/master/machine-learning/tensorflow/Label%20Image.ipynb) in Jupyter Notebook

## Step 3. Add Tensorflow Mobile on the Project
There are two ways to add Tensorflow Mobile into your projects.

1. Download the C++ source code and make the library.
2. Use the pre-build library from jCenter.

The second way is most likely you add other libraries on android projects, so I will use this method in this article as well.

``` js
// add this line in build.gradle
dependencies {
    compile 'org.tensorflow:tensorflow-android:1.2.0'
}
```

### TensorFlowInferenceInterface
The main class of TensorFlow Mobile is TensorFlowInferenceInterface. It only supports load protobuf model from assets folder. Therefore, the file generated from step 2 have to put under `./project/app/src/main/assets` as well as the labels.txt.

> To create the folder. right click upon the project on Android Studio. Then go to `New > Folder > Asset Folder`.

## Step 4. Run the classification
``` java
int CLASS_SIZE = 1001; // MobileNet has 1000 classes + 1 zero-index
int IMAGE_SIZE = 224; 
String INPUT_NAME = "input";
String OUTPUT_NAME = "MobilenetV1/Predictions/Reshape_1";
String[] OUTPUT_NAMES = {OUTPUT_NAME};

// load the model
TensorFlowInferenceInterface tfii = new TensorFlowInferenceInterface(context.getAssets(), "mobilenet_v1.pb");

// input a image for classification
tfii.feed(INPUT_NAME, imageFloats, 1, IMAGE_SIZE, IMAGE_SIZE, 3);

// run the classification as session.run() in python.
tfii.run(OUTPUT_NAMES, false);

// get output variables which are the probabilities of the classes.
float[] outputs = new float[CLASS_SIZE];
tfii.fetch(OUTPUT_NAME, outputs);

// than do argmax the outputs to get the top of the highest probabilities of the classes.
```

<img src="https://github.com/wadehuang36/tensorflow_mobilenet_android_example/blob/master/screenshots/screenshot-1.png?raw=true" style="max-width:300px">
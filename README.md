
# CNN (YOLOv3) for object detection and classification

# Overview
This is a project based on deep Convolutional Neural
Network (CNN). I will explore the state-of-the-art CNN with ResNet, YOLO, and
InceptionNet using dataset from [Open Images Challenge
2018](https://storage.googleapis.com/openimages/web/index.html). The goal is to
familiarize with hyperparameter tuning, network architecture, and data
visualization. I will use pre-trained weights to perform "transfer
learning" on different classes, which may involve "fine-tuning" the model
(i.e., unfreezing the weights in previous layers) since
the final classes and the dataset to pretrain the model may be different.
Usually fine-tuning on the same dataset only involves unfreezing the last layer
(for darknet using unchanged cfg file, stopbackward=1 and without "partial"). 

(update 9/28/2018: darknet has pretrained weights on Open Image dataset, so I will test 
the pretrained model before any fine-tuning) 

# Setting up Docker-enabled Google Cloud Platform ([GCP](https://console.cloud.google.com/home/))
* Create VM instances (with preemptibilty to save cost) with Container-Optimized OS (COS) 66-10452.109.0 stable. 
* Add an additional *Standard Persistent Disk* to share with multiple VM by [mounting](https://cloud.google.com/compute/docs/disks/add-persistent-disk#create_disk) the disk `openimage` under the path `/mnt/disks/`.

# Data acquisition
* Method 1 (preferred): [Bigquery](https://bigquery.cloud.google.com/table/)
  * At "openimagesChallenge" click "Create new dataset" and enter `open_images_yolov3` for "Dataset ID".
    (caution: unselect "Use Legacy SQL" to enable "Standard SQL Dialect").
  * Run the query for keras-yolo3 annotation file:
    ```
    SELECT im.original_url, ab.x_min, ab.x_max, ab.y_min, ab.y_max, ab.label_name
    FROM `bigquery-public-data.open_images.annotations_bbox` as ab
    JOIN `bigquery-public-data.open_images.images` as im
    ON ab.image_id = im.image_id
    ```

    click "Save as table" and set "Destination table" to "open_images_annotation". Now we can use PySpark to set the 
    [configuration](https://cloud.google.com/dataproc/docs/tutorials/bigquery-sparkml) for importing data from BigQUery:

    ``` python
    from datetime import datetime
    from pyspark.context import SparkContext
    from pyspark.ml.linalg import Vectors
    from pyspark.sql.session import SparkSession

    sc = SparkContext()
    spark = SparkSession(sc)
    bucket = spark._jsc.hadoopConfiguration().get("fs.gs.system.bucket")
    project = spark._jsc.hadoopConfiguration().get("fs.gs.project.id")
    todays_date = datetime.strftime(datetime.today(), "%Y-%m-%d-%H-%M-%S")
    input_directory = "gs://{}/tmp/open_images-{}".format(bucket, todays_date)
    conf = {
        "mapred.bq.project.id": project,
        "mapred.bq.gcs.bucket": bucket,
        "mapred.bq.temp.gcs.path": input_directory,
        "mapred.bq.input.project.id": project,
        "mapred.bq.input.dataset.id": "open_images_yolov3",
        "mapred.bq.input.table.id": "open_images_annotation",
        }
    # Read the data from BigQuery into Spark as an RDD.
    table_data = spark.sparkContext.newAPIHadoopRDD(
        "com.google.cloud.hadoop.io.bigquery.JsonTextBigQueryInputFormat",
        "org.apache.hadoop.io.LongWritable",
        "com.google.gson.JsonObject",
        conf=conf)
    ``` 

* Method 2: Manual download
  * Download the [Open Image dataset](https://www.figure-eight.com/dataset/open-images-annotated-with-bounding-boxes/) to the openimage disk by simply using `wget`."

  * To `unzip` in COS, I run the docker image from GCP [cos-toolbox](gcr.io/google-containers/toolbox) with `-v` to mount local directory into the `toolbox` container.
(*update: I could've used a OS-based docker image with necessary daemon and pre-installed python version, which is the more direct and compact approach. 
The current approach may save a bit on memory.*)

  > docker run --name toolbox -v /mnt/disks/openimage:/mnt -d gcr.io/google-containers/toolbox tail -f /dev/null
  * Run an interactive bash session in the `toolbox` container

  > docker exec -it toolbox bash

  * `unzip` the data in the mounted directory `/mnt` in the container, pointing to the persistent disk /mnt/disks/openimage (mounted on my local VM directory).

  > unzip train00.zip

# Setting up Python packages with Docker
*  Create `Dockerfile` to manage the python environment for this project. 

> cat Dockerfile

```
# Use the official Python runtime as a parent image
FROM python:3.5.6-stretch
# Set the working directory to /app in the containerA
WORKDIR /app
# Copy the current local directory contents into the container at /app
ADD . /app
# Install the project related packages
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
RUN python get-pip.py
RUN pip2 install --trusted-host pypi.python.org -r requirements.txt
```

> cat requirements.txt

```
opencv-python \ keras \ pandas \ numpy==1.14.5 \ cython \ tensorflow \ matplotlib \ h5py \ scipy \ pydot \ argparse \ cython
```

* Build the `Dockerfile` to produce an image with name `openimage` in the repository `gchen0119` with the tag `python3.5.6-stretch`

> docker build .

> docker tag <IMAGE_ID> gchen0119/openimage:python-3.5.6-stretch

* Login my public registry 

> docker  login --username=gchen0119 \-\-email=gchen0119@gmail.com

* Push the image onto my Docker public registry [gchen0119](https://hub.docker.com/r/gchen0119) 
(check out my Random Notes on how to push to GCP `Container Registry`)
> docker push gchen0119/python-openimage:python-3.5.6-stretch


# Start Python in the container

* Detach and run the image in the `python-openimage` container (use -v to mount local dir in the container)

> docker run -v /mnt/disks/openimage:/mnt --name python-openimage -d gchen0119/openimage tail -f /dev/null

* Enter the `python-openimage` container interactively in the bash session

> docker exec -it python-openimage bash

* Start using python in the container and import all the necessary packages


# Recap YOLOv2 and YOLO9000: 

* YOLOv2 (19-conv layers): 
    * Dimension clusters using k-means distance measure with IoU to define anchor boxes/priors.  
    * Anchor boxes obtain the most frequent aspect ratios for all objects, therefore learning these boxes accelerate the training.
    * Anchor boxes also allow multiple overlapped objects to be detected.
    * Bounding box center coordinate predictions are constrained in the grid cell by using sigmoid/logistic function ranging between 0 and 1, and accelerating the training.
    * Classification and detection are done on different datasets and trained sequentially with a single softmax at the output layer.
   
* YOLO9000: 
    * Combine YOLOv2 with WordTree, this resolves the joint training for classification and detection on combined datasets with multiple softmax concatenated at the output layer.


# YOLOv3 Model Details

* YOLOv3 (53-conv layers):
    * Use independent logistic classifiers for each class 
      (i.e. multilabel classification, e.g., Women and Person).
    * Predictions across 3 different scales (similar to Figure 3 in 
      [Feature Pyramid Network (FPN)](https://arxiv.org/abs/1612.03144)).
        * <p align="center">
            <img src="feature_pyramid.png"/>
          </p>
        * FPN essentially connects the low level convolutional layers
          (with stronger localized activation) to the same-sized (i.e., same "stage") 
          high level upsampled layers (with stronger semantics). This is called 
          "lateral" connection by adding the two feature maps. 
        * For YOLOv3, a "route" at two convolutional layers before the output
	  layer is upsampled and concatenated with an even earlier layer (with 
          localized feature activation), e.g., the "[route] -1,61" appearing in the 
          "cfg" file is doing the "lateral" concatenation as FPN. The latter
          route will concatenate with even earlier layers, which is exactly
          the "Top-down pathway" laterally merging with "Bottom-up pathway" of FPN.
            * Upsampling/deconvolution with fully convolutional layers to extract
              features from earlier layers for a meaningful semantic information.
    * Skip connection/identity map from an earlier layer to a later layer 
      (i.e., residual layer) act to prevent the vanishing gradient problem.

# Environment setup and testing for darknet with Keras
* The YOLOv3 is developed by [pjreddie](https://pjreddie.com/darknet/yolo/)
  with [source codes](https://github.com/pjreddie/darknet.git). 

[//]: # "Note: one can simply build YOLOv3 related packages with Dockerfile via [madhawav's](https://github.com/madhawav/YOLO3-4-Py/blob/master/docker/)."

[//]: # "The python wrapper is from [madhawav](https://github.com/madhawav/YOLO3-4-Py) and on [pypi](https://pypi.org/project/yolo34py/#description)."

```python
# Keras functions
# Try these CNN later on
#from keras.applications import ResNet50
#from keras.applications import InceptionV3
#from keras.applications import Xception # TensorFlow ONLY
#from keras.applications import VGG16
#from keras.applications import VGG19
#from keras.applications import imagenet_utils
#from keras.applications.inception_v3 import preprocess_input
from keras.preprocessing.image import img_to_array
from keras.preprocessing.image import load_img
from keras.models import load_model, Model
from keras.layers import Input, Lambda, Conv2D
from keras import backend as K
import tensorflow as tf
import numpy as np
import argparse
import sys
# YOLOv3 model functions
sys.path.append('/mnt/python/keras-yolo3/')
from yolo3.model import yolo_eval # Evaluate YOLO model on given input and return filtered boxes.
from pydarknet import Detector, Image # YOLOv3 package
import cv2 # OpenCV, OpenCV 3.4.1 will fail with darknet 
import yolo
import yolo_video

%matplotlib inline
```

* Download the YOLOv3 weights for Open Images 

> wget https://pjreddie.com/media/files/yolov3-openimages.weights 

* Use the Keras implementation of YOLOv3 (Tensorflow backend) by
  [qqwweee](https://github.com/qqwweee/keras-yolo3) to build the model, 
  and save as `yolo-openimages.h5` (make sure to comment either the train 
  or test `batch` and `subdivision`).

> python convert.py yolov3-openimages.cfg yolov3-openimages.weights model_data/yolo-openimages.h5 


* Loading a pretrained model 

```python
yolo_model = load_model("model_data/yolo-openimages.h5") # load the model
yolo_model.summary() # show a summary of the model layers
```

* Object detection (uses yolo.eval() to return bounding boxes, yolo.head() returns the bounding box dimensions and feature probability)

```python
# configure the default to YOLOv3 on Open Images
yolo.YOLO._defaults['model_path']='model_data/yolo-openimages.h5'
yolo.YOLO._defaults['classes_path']='model_data/openimages.names'
yolo.YOLO._defaults['anchors_path']='model_data/yolo_anchors.txt'
yolo_video.detect_img(yolo.YOLO()) # comment r_image.show(), and add r_image.save(filename) to yolo_video.py
# yolo_video.detect_img() uses yolo.detect_image(), with additional ability to input multiple images on the fly
```

* Verify the results by darknet  
```c
./darknet detector test cfg/openimages.data cfg/yolov3-openimages.cfg yolov3-openimages.weights
```

* Test image result, notice multiple labels around the dog showing dependencies between classes!
![alt](doggy.jpg)

# Test the mAP (mean average precision) of current Open Image data


# Start fine-tuning with Open Images data

* Need to create a file `openimages_annotation.py` following `voc_annotation.py` to use train.py on `train.txt` with lines as 
  `path/to/img.jpg xmin1,ymin1,xmax1,ymax1,class_id1 xmin2,ymin2,xmax2,ymax2,class_id2`.

* `train.py` does a freeze retraining and then a unfreeze retraining if mAP aren't that good.

# TO BE CONTINUED...
--------------------------------------------------------------------------------------
# Random Notes
* My [notes](https://github.com/ginochen/macHome/tree/master/notes/c/darknet) on darknet source code (C and Keras).

* Transferring data between my laptop and the VM instance using the save ssh private key

> scp -i ~/.ssh/google_compute_engine /path/to/file gino@<REMOTE_IP>

where `<REMOTE_IP>` is assigned at the start of a VM instance.

* Transferring data between my laptop and google bucket 

> gcloud compute ssh gchen@<LOCAL_IP> --command='gsutil cp /path/to/file gs://mybucket' 

where `<LOCAL_IP>` can be located by `ifconfig` on macbook or `ip addr show` on linux.

* In order to push the image to [Container Registry](https://cloud.google.com/container-registry/docs/pushing-and-pulling#pushing_an_image_to_a_registry)  hosted by google
on `gcr.io`, first login to google cloud shell and do

> gcloud config set project openimagechallenge 

where `openimagechallenge` is the `Project ID` shown on the console. 

Then login to `gcr.io` to get authorization 

> gcloud auth login

and follow the steps to authenticate.

Then obtain the access-token for docker

> gcloud auth print-access-token

Copy the access-token and go to the COS VM instance

> docker login -u oauth2accesstoken -p "<paste-the-access-token-here\>" https://gcr.io

Finally `push` the image into the google cloud Container Registry (just like Docker Hub)

> docker push <HOSTNAME\>/<PROJECT_ID\>/<IMAGENAME\>:TAG

In my case

> docker push gcr.io/openimageschallenge/openimage:python3.5.6-stretch

Check out the image save on the GCP registry by going the left-hand-side of the panel and click "Contianer Registry".

Note access token changes every time, so always login the cloud shell to `print-access-token`.

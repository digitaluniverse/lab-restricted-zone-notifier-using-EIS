# Implementing an Classifier and Preprocessing Trigger based Intel® Edge Insights (EIS) Software

This lab shows the steps that an application developer will need to implement to create a video analytics solution on the Edge Insights Software framework.

## Reference Implementations on the Intel Development Zone

The Intel Developer Zones has many reference implementation applications, sample code snippets and whitepapers to help you start building with Intel technology.

On the [Industrial Reference Implementations and Code Samples](https://software.intel.com/en-us/industrial) page, you can browser difference ready-made applications. We will be porting the [Restricted Zone Notifier](https://software.intel.com/en-us/iot/reference-implementations/restricted-zone-notifier) to the Edge Insights Software framework.

The source code for the Restricted Zone Notifier can be found on Github.
https://github.com/intel-iot-devkit/restricted-zone-notifier-python.

## Description of the Restricted Zone Notifier that will be Ported to EIS

First let's look at the reference implementation of the Restricted Zone Notifier. This is an OpenVino application which determines whether a worker is in an unsafe or restricted zone. The classification results are sent to an MQTT broker and published for third-party applications.

## Steps to Port Application to EIS

1. Create the Restricted Zone Notifier Application Configuration JSON file
2. What is a directory called restricted Zone Notifier in the classifier directory
3. Create a Python file for the user defined classifier algorithm
4. Deploy application

###

First, let us have a look on the below code flow for the restricted zone notifier application.

![](images/flowchart.png)

**Now**, Lets understand how we can convert this python based RI to EIS framework step by step.

### Setup Environmental Variables

For our convenience, let's set an environmental variable called **EIS_HOME** to refer to the root level directory of the Edge Insights Software.

**During the workshop be sure to check this location or ask your instructor. **

```bash
export EIS_HOME=/home/eis/IEdgeInsights-lab-restricted-zone-notifier
```

### Create Directories for the New EIS Application

First, let's create the directory for the new user-defined classification function and create an empty ****init**.py** file to contain our implementation.

```bash
mkdir $EIS_HOME/algos/dpm/classification/classifiers/restrictedzonenotifier
touch $EIS_HOME/algos/dpm/classification/classifiers/restrictedzonenotifier/__init__.py
```

### Create the Application Configuration JSON file

In the **$EIS_HOME/docker_setup/config/algo_config** create a file named **restricted_zone_notifier.json**. This file will contain a JSON object that defines the video sources, preprocessing trigger location and the classifier function location.



When launching the reference implementation normally parameters such as the model to use, any library plugins, the hardware to run inference on, and the location of the input file are set on the command line:

![](images/rzn_input_1.png)

We will set these configuration parameters in the restricted_zone_notifier.json application configuration file in the **video_file**, **model_xml**, **model_bin**, and **device** fields. 

We will also set the trigger alogorithm and the classification alogrithm to be used in the solution in this configuration file. 

The reference implementation does not have any data pre-processing so we will use the **bypass_trigger** which sends all incoming frames to the classification engine. 

For the classification module we will point to the currently empty **restrictedzonenotifier** folder that we created earlier. 


Create the .json file:

```bash
gedit $EIS_HOME/docker_setup/config/algo_config/restricted_zone_notifier.json
```

Next copy and paste this text into the newly created file:

```JSON
{
    "machine_id": "tool_2",
    "trigger_threads": 1,
    "queue_size" : 10,
    "data_ingestion_manager": {
        "ingestors": {
            "video_file": {
                "video_file": "./test_videos/worker-zone-detection.mp4",
                "encoding": {
                    "type": "jpg",
                    "level": 100
                },
                "img_store_type": "inmemory_persistent",
                "loop_video": true,
                "poll_interval": 0.2
            }
        }
    },
    "triggers": {
        "restricted_zone_notifier_trigger": {
            "training_mode": false
        }
    },
    "classification": {
        "max_workers": 1,
        "classifiers": {
            "restrictedzonenotifier": {
                "trigger": [
                    "restricted_zone_notifier_trigger"
                ],
                "config": {
                    "model_xml": "./algos/algo_config/restricted_zone_notifier/person-detection-retail-0013.xml",
                    "model_bin": "./algos/algo_config/restricted_zone_notifier/person-detection-retail-0013.bin",
                    "device": "CPU"
                }
            }
        }
    }
}
```

### Download Required Video files and Inference Models

Notice that the **data_ingestion_manager** has one ingestor defined and it is a video file located at **"./test_videos/worker-zone-detection.mp4"**

You will need to download this file and place it in the following directory:
```
cd $EIS_HOME/docker_setup/test_videos
wget https://raw.githubusercontent.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/master/videos/worker-zone-detection.mp4
```

You will also need to download the OpenVINO inference models and label files **person-detection-retail-0013.xml** and **person-detection-retail-0013.bin**

```bash
mkdir $EIS_HOME/docker_setup/config/algo_config/restricted_zone_notifier
cd    $EIS_HOME/docker_setup/config/algo_config/restricted_zone_notifier

wget https://github.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/blob/master/Models/person-detection-retail-0013.bin
wget https://raw.githubusercontent.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/master/Models/person-detection-retail-0013.xml

```

### Creating the Classifier algorithm 

In EIS the classifier algorithm has 3 mehthods: 


`__init__` : The method used for initialization and loading the intermediate representation model into the plugin. 
 
`classify` : The method used for inferencing and capturing the inference output.
 
`ssd_out`: The method used fpr parsing the classification output. 

We will use the main python script from the [reference implementation](https://github.com/intel-iot-devkit/restricted-zone-notifier-python/blob/master/application/restricted_zone_notifier.py) as a basis in creating these three methods and add them to out **__init__.py** file.

### Import modules and create Classifier class

First we will import all python modules that will be used in the classifier algorithm and create the main **Classifier** class which will contain our methods:

First open our **__init__.py** file:

```bash
gedit $EIS_HOME/algos/dpm/classification/classifiers/restrictedzonenotifier/__init__.py
```

and copy the following code at the top:

```python
import os
import sys
import logging
import cv2
import time
import numpy as np
import json
from collections import namedtuple
from algos.dpm.defect import Defect
from algos.dpm.display_info import DisplayInfo
from openvino.inference_engine import IENetwork, IEPlugin
MyStruct = namedtuple("assemblyinfo", "safe")
INFO = MyStruct(True)
PERSON_DETECTED = 1
class Classifier:
```
### Create __init__ method

To create the `__init__` method we will check that the model files exist, load the plugin for CPU, load the CPU extension libraries, and check that the layers of the model are supported.

Paste is the following code into our **__init__.py** file:

```python
    def __init__(self, model_xml, model_bin, device):
        """Constructor
        Parameters
        ----------
        model_xml      : TF model xml file
        model_bin      : TF model bin file
        device         : Run time device [CPU/GPU/MYRIAD]
            Classifier configuration
        """

        self.log = logging.getLogger('PEOPLE_DETECTION')

        # Assert all input parameters exist
        assert os.path.exists(model_xml), \
            'Tensorflow model missing: {}'.format(model_xml)
        assert os.path.exists(model_bin), \
            'Tensorflow model bin file missing: {}'.format(model_bin)

        # Load OpenVINO model
        #loading plugin for CPU
        self.plugin = IEPlugin(device=device.upper(), plugin_dirs="")

        if 'CPU' in device:
            self.plugin.add_cpu_extension("/opt/intel/openvino/inference_engine/lib/intel64/libcpu_extension_sse4.so")

        self.net = IENetwork(model=model_xml, weights=model_bin)

        if device.upper() == "CPU":
            supported_layers = self.plugin.get_supported_layers(self.net)
            not_supported_layers = [l for l in self.net.layers.keys() if l not
                                    in supported_layers]
            if len(not_supported_layers) != 0:
                self.log.debug('ERROR: Following layers are not supported by \
                                the plugin for specified device {}:\n \
                                {}'.format(self.plugin.device,
                                           ', '.join(not_supported_layers)))

        assert len(self.net.inputs.keys()) == 1, \
            'Sample supports only single input topologies'
        assert len(self.net.outputs) == 1, \
            'Sample supports only single output topologies'

        self.input_blob = next(iter(self.net.inputs))
        self.output_blob = next(iter(self.net.outputs))
        self.net.batch_size = 1  # change to enable batch loading
        self.exec_net = self.plugin.load(network=self.net)
   ```


### Create Classify method 

To create the `classify` method we will use the section of the main function loop that runs the single shot detector on each frame as as well as the section of code that writes the alerts out to the screen as a basis: 

```python
 def classify(self, frame_num, img, user_data):
        """Classify the given image.

        Parameters
        ----------
        frame_num : int
            Frame count since the start signal was received from the trigger
        img : NumPy Array
            Image to classify
        user_data : Object
            Extra data passed forward from the trigger

        Returns
        -------
            Returns dissplay info and detected coordinates
        """
        self.log.info('Classify')
        d_info = []
        p_detect = []
        frame_count=+1

        initial_wh = [img.shape[1], img.shape[0]]
        n,c,h,w =  self.net.inputs[self.input_blob].shape

        roi_x,roi_y, roi_w,roi_h = [0,0,0,0]

        if roi_x <= 0 or roi_y <= 0:
           roi_x = 0
           roi_y = 0
        if roi_w <= 0:
            roi_w = img.shape[1]
        if roi_h <= 0:
             roi_h = img.shape[0]

        cv2.rectangle(img, (roi_x, roi_y),
                      (roi_x + roi_w, roi_y + roi_h), (0, 0, 255), 2)
        selected_region = [roi_x, roi_y, roi_w, roi_h]

        in_frame_fd = cv2.resize(img, (w, h))
        # Change data layout from HWC to CHW
        in_frame_fd = in_frame_fd.transpose((2, 0, 1))
        in_frame_fd = in_frame_fd.reshape((n, c, h, w))

        # Start asynchronous inference for specified request.
        inf_start = time.time()
        self.exec_net.start_async(request_id=0, inputs={self.input_blob:in_frame_fd})
        self.infer_status=self.exec_net.requests[0].wait()
        det_time = time.time() - inf_start

        res = self.exec_net.requests[0].outputs[self.output_blob]
        # Parse SSD output
        safe,person=self.ssd_out(res, initial_wh, selected_region)

        if person:
            x,y,x1,y1 = [person[0][i] for i in (0,1,2,3)]
            p_detect.append(Defect(PERSON_DETECTED, (x, y), (x1, y1)))

        # Draw performance stats
        inf_time_message = "Inference time: {:.3f} ms".format(det_time * 1000)
        throughput = "Throughput: {:.3f} fps".format(1000*frame_count/(det_time*1000))

        d_info.append(DisplayInfo(inf_time_message, 0))
        d_info.append(DisplayInfo(throughput, 0))
        #if not safe display HIGH [priority: 2] alert string
        if p_detect:
            warning = "HUMAN IN ASSEMBLY AREA: PAUSE THE MACHINE!"
            d_info.append(DisplayInfo('Worker Safe: False', 2))
            d_info.append(DisplayInfo(warning, 2))
        else:
            d_info.append(DisplayInfo('Worker Safe: True', 0))

        return d_info, p_detect
 ```


### 3. Capture frames from input file

- In the python based code, the frames are captured from image/video using cv2.Videocapture.

* In EIS framework, the fframe capture is done using the **Trigger** module.The trigger module in the EIS stack receives the captured frames from the VideoIngestion module. The purpose of the trigger algorithm is to select frames of interest from the camera stream and pass them to the VideoAnalytics module. The algorithm to use in the trigger modul depends on the use case. For people monitoring you might need a classifier algorithm to execute on all frames whereas a use case like the sample application “pcbdemo” would need the classifier to execute on only the frames where the PCB board is in the center of the frame.

* After the video frames are processed by the trigger they are registered by the register_trigger_callback() method to be passed to the classifier.

The section of code in main.py that deals with the video frame input is ported to the resttricted zone notifier trigger script.

![](images/rzn_trigger_1.png)
![](images/arrow.png)
![](images/rzn_trigger_2.png)

### 4. Run Classification and parse results

After the trigger selects a frame it is sent to the classifier to run through the model and generate the results needed.
To set up the classifier we use the code from main.py which defines the Single Shot Detector (SSD) model and port it over to `~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/classification/classifiers/restrictedzonenotifier/__init__.py`

![](images/rzn_ssd_out_1.png)
![](images/arrow.png)
![](images/rzn_ssd_out_2.png)

### 5. Update Status and Alert information

- The status and alert information is also included in the same classifier `__init__.py`. This classify method returns defect information as well as statistics about the inference time and throughput of the data pipeline.

![](images/rzn_output_1.png)
![](images/arrow.png)
![](images/rzn_output_2.png)

### 6. Messaging Thread

The Python based reference implementation messaging thread publishes MQTT messages to Server to display the output.

In the EIS framework the messages are published over OPC/UA by the Data Agent Service. We will use an OPC/UA client to view those messages. This OPC/UA client is located in ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer and does not need to be customized for this application.

You should now have a better idea of how an existing code base can be converted to Classifier and Trigger based Intel® Edge Insights (EIS) Software. In the next lab we will implement all of these modules and run the restricted zone notifier using EIS.

## Next Lab

[Deploying Restricted Zone Notifier using Edge Insights Software framework](./lab_restricted_zone_notifier.md)

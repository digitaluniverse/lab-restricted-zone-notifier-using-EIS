## Deploying Restricted Zone Notifier using Intel® Edge Insights Software framework
### Lab Overview
In our previous Lab, we have successfully ran the pcbdemo application using the Intel® Edge Insights Framework. Now, we will deploy a Restricted zone notifier reference implementation using Intel® Edge Insights Framework.

### Steps to Complete this lab:
- Create custom trigger that will send frames to the classifier when a person is detected in frame.
- Create a custom classifier to load the frame from the trigger.
- Create a custom .json configuration file to run the code.
- Customize visualizer application in order to view the resulting output.
- Setup OPC/UA server to receive warning messages form the classifier.
- Run the application using different Intel's Accelerators(CPU/GPU/MYRIAD).


## Step-1 : Develope a Custom trigger
The purpose of trigger algorithm is to select frames of interest from the camera stream. The algorithms depends on the use case, people monitoring might need classifier algorithms to execute on all frames from the camera whereas a use case like the sample application “pcbdemo” would need the classifier to execute on only the frames where the PCB board is in the center of the frame

- To create trigger a custom trigger go to ***~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/triggers*** and create a trigger file with name restricted_zone_notifier_trigger.py

- Enter the following commands in terminal to complete this.

    ```bash
  cd ~/Workshop/IEdgeInsights-v1.5LTS//algo/dpm/triggers/
  sudo gedit restricted_zone_notifier_trigger.py
  ```
- Copy the below code snippet to restricted_zone_notifier_trigger.py

  ```python
  import logging
  import cv2
  import numpy as np
  from . import BaseTrigger

  class Trigger(BaseTrigger):

    def __init__(self, training_mode):
        """Constructor.
        """

        super(Trigger, self).__init__()
        self.log = logging.getLogger(__name__)
        # Flag to lock trigger from forwarding frames to classifier
        self.training_mode = training_mode
        self.count = 0
        self.startSignal = True

    def get_supported_ingestors(self):
        return ['video', 'video_file']

    def on_data(self, ingestor, data):
        """Process video frames as they are received and call the callback
        registered by the `register_trigger_callback()` method if the frame
        should trigger the execution of the classifier.

        Parameters
        ----------
        ingestor : str
            String name of the ingestor which received the data
        data : tuple
            Tuple of (camera serial number, camera frame)
        """
        self.count = self.count + 1
        if self.training_mode is True:
            self.send_start_signal(data, -1)
            cv2.imwrite("./frames/"+str(self.count)+".png", data[1])
        else:   
            # Send trigger start signal and send frame to classifier
            if self.startSignal:
                self.send_start_signal(data, -1)
                self.startSignal = False
            self.log.info("Sending frame")
            self.send_data(data, 1)
    ```



## Step-2 : Develop a Custom classifier

The purpose of classifier algorithm is to load the frame from the trigger into the SSD model to detect location of people and a secondary algorithm that detects if people are inside the restricted zone. If people are detected in the restricted zone a warning should be sent to the output frame and a warning should be sent over OPC/UA to the operator.

- To create trigger a custom classifier go to **~/Workshop/IEdgeInsights-v1.5LTS/algo/dpm/classification/classifiers**  
- Create a folder with name  **"restrictedzonenotifier"** and create a file with name ```__init__.py```
- Enter the following commands in terminal to complete this.

  ```bash
    cd ~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/classification/classifiers/
    mkdir restrictedzonenotifier && cd restrictedzonenotifier
    sudo gedit __init__.py
    ```
- Copy the below classifier code snippet to the newly created ```__init__.py```.

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

        def ssd_out(self,res, initial_wh, selected_region):
            """
            Parse SSD output.

            :param res: Detection results
            :param args: Parsed arguments
            :param initial_wh: Initial width and height of the frame
            :param selected_region: Selected region coordinates
            :return: safe,person  
            """
            global INFO
            person = []
            INFO = INFO._replace(safe=True)

            for obj in res[0][0]:
                # Draw objects only when probability is more than specified threshold
                if obj[2] > 0.5:
                    xmin = int(obj[3] * initial_wh[0])
                    ymin = int(obj[4] * initial_wh[1])
                    xmax = int(obj[5] * initial_wh[0])
                    ymax = int(obj[6] * initial_wh[1])
                    person.append([xmin, ymin, xmax, ymax])

            for p in person:
                # area_of_person gives area of the detected person
                area_of_person = (p[2] - p[0]) * (p[3] - p[1])
                x_max = max(p[0], selected_region[0])
                x_min = min(p[2], selected_region[0] + selected_region[2])
                y_min = min(p[3], selected_region[1] + selected_region[3])
                y_max = max(p[1], selected_region[1])
                point_x = x_min - x_max
                point_y = y_min - y_max
                # area_of_intersection gives area of intersection of the
                # detected person and the selected area
                area_of_intersection = point_x * point_y
                if point_x < 0 or point_y < 0:
                    continue
                else:
                    if area_of_person > area_of_intersection:
                        # assembly line area flags
                        INFO = INFO._replace(safe=True)
                    else:
                        # assembly line area flags
                        INFO = INFO._replace(safe=False)
            return INFO.safe,person

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

## Step-3 : Customizing the configuration

In this section we will create a custom JSON  configuration file to integrate the custom trigger and custom classifiers developed in previous steps.

- To create a custom configuration file, go to **~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/config/algo_config/**
- Create a JSON file with name **restricted_zone_notifier.json**

- Enter the following commands in terminal to complete this.

  ```bash
  cd ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/config/algo_config/
  sudo gedit restricted_zone_notifier.json

  ```
  - Copy the below configuration details to **restricted_zone_notifier.json**

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

To integrate the developed restricted zone notifier demo, following files to be updated with restricted_zone_notifier.json file:

- IEdgeinsights/docker_setup/.env
- IEdgeinsights/VideoIngestion/VideoIngestion.py
- IEdgeinsights/DataAnalytics/PointDataAnalytics/classifer_setup.py

#### Update the environment
- To complete this, run the following commands in the terminal

    ```bash
    cd  ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup
    sudo gedit .env
    ```
- Find and replace the name of the configuration file ```factory_pcbdemo.json ``` with ```restricted_zone_notifier.json``` .
- Make sure that DEV_MODE = false 

#### Update VideoIngestion module
- To complete this, run the following commands in the terminal

    ```bash
    cd  ~/Workshop/IEdgeInsights-v1.5LTS/VideoIngestion
    sudo gedit VideoIngestion.py
    ```
- Find and replace the name of the configuration file ```factory_pcbdemo.json ``` with ```restricted_zone_notifier.json``` .


#### Update classifier setup
- To complete this, run the following commands in the terminal

    ```bash
    cd ~/Workshop/IEdgeInsights-v1.5LTS/DataAnalytics/PointDataAnalytics
    sudo gedit classifier_startup.py
    ```
- Find and replace the name of the configuration file ```factory_pcbdemo.json ``` with ```restricted_zone_notifier.json``` .


## Step-4: Customize visualizer application

For visualizing the results of the video analytics, the a visualizer app is available with the IEdgeInsights software framework. This is a sample app which uses the OPC-UA client for receiving the analytics results and the GRPC client for receiving the image frames and do a simple visualization.

To customize the visualizer application(visualizer.py) modify the **config .json** file available in  **~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer/** path.

- To complete this task run the following commands to complete this task
    ```bash
    cd ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer
    sudo gedit config.json
    ```
and replace the contents of the config.json with the below configuration details to view the result in a single window

    ```JSON
    {
      "output_streams": [
        "stream1_results"
      ]
    }
    ```

### Step-5 : Download video 
We will be using a test video from the Intel IoT DevKit repository. It shows workers entering a simulated restricted zone and will be used to test the EIS pipeline.

```bash
cd ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/test_videos
wget https://github.com/intel-iot-devkit/sample-videos/raw/master/worker-zone-detection.mp4
```

### Step-6: Download Model 
The ia_data_analytics docker container which runs the classification using openVINO requires a specific person detection model to detect when workers enter the restricted zone. Usually you would use the OpenVINO model downloader to get the models but we will just pull them from the workshop Github repo.  The model files need to be copied into the docker setup path so that the ia_data_analytics container can locate it:

```bash
cd ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/config/algo_config
mkdir restricted_zone_notifier
cd restricted_zone_notifier
wget https://github.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/raw/master/Models/person-detection-retail-0013-fp16.bin
wget https://github.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/raw/master/Models/person-detection-retail-0013-fp16.xml
wget https://github.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/raw/master/Models/person-detection-retail-0013.bin
wget https://github.com/SSG-DRD-IOT/lab-restricted-zone-notifier-using-EIS/raw/master/Models/person-detection-retail-0013.xml
```

### Step-7 : Build and Run the Application
Run the following commands to build and run the customized restricted zone notifier apllication using EIS.

#### Generate certificates:
- Run the following commands

    ```bash
    cd ~/Workshop/IEdgeInsights-v1.5LTS/cert-tool
    python3 main.py
    ```

#### Provision the secrets to Vault:
This will take the inputs from [docker_setup/config/provision_config.json](docker_setup/config/provision_config.json) & read the cert-tool generated Certificates and save it securely by storing it in the Hashicorp Vault.
- Run the following commands

    ```bash
    cd ~/IEdgeinsights/docker_setup/
    sudo make provision CERT_PATH=../cert-tool/Certificates/
    sudo make install CERT_PATH=../cert-tool/Certificates/
    ```
- Next check to see if the EIS pipeline is running properly: 

    ```bash
    tail -f /opt/intel/iei/logs/consolidatedLogs/iei.log
    ```


#### IEI visualizer setup

- Run the following commands to run the customized restricted zone notifier apllication using EIS.

    ```bash
    sudo make distlibs
    cd ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer
    sudo make build
    sudo make run CERT_PATH=~/Workshop/IEdgeInsights-v1.5LTS/cert-tool/Certificates/ HOST=localhost IMAGE_DIR=/opt/intel/iei/saved_images DISPLAY_IMG=true
    ```

 On successful execution, the application sends a warning message when a person is detected in the restricted zone. The output looks like below screenshot.

 ![](images/restricted_zone_notifier_result.png)
 
 Notice also the JSON object which is written to the terminal: 
 
 ```json
 Received message: {"Measurement":"stream1_results","Channels":3.0,"Height":1080.0,"ImgHandle":"persist_bc736c96","Width":1920.0,"cam_sn":"camera-serial-number","defects":"[{\"type\":1,\"tl\":[436,237],\"br\":[686,959]}]","display_info":"[{\"info\":\"Inferencetime:42.387ms\",\"priority\":0},{\"info\":\"Throughput:23.592fps\",\"priority\":0},{\"info\":\"WorkerSafe:False\",\"priority\":2},{\"info\":\"HUMANINASSEMBLYAREA:PAUSETHEMACHINE!\",\"priority\":2}]","encoding":"jpg","idx":59979.0,"image_id":"600bef07-3915-42e3-818b-c0fe76d2c40a","machine_id":"tool_2","part_id":"e0550d60-3a4b-4f2d-82fc-98fbebdf8626","timestamp":1567196325.5299864,"influx_ts":1567196325532124746
```

Here we can see the alert status and performance of the application. 

## Step-8 : Optimizing Application
Now, we will explore ways of improving performance of the application.

## Visualizer 
Fetching individual frames from the image store via the Data Agent can slow down the overall performance of the application. In a industrial safty application such as the zone notifier the alert output will be used by some other system to issue an alert or pause the machinery - and so there is no need for an operator to view the video in real time. 

Let's increase the through put of our application by disabling the video frames from being fetched and displayed:

```bash
cd ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer
    sudo make build
    sudo make run CERT_PATH=~/Workshop/IEdgeInsights-v1.5LTS/cert-tool/Certificates/ HOST=localhost IMAGE_DIR=/opt/intel/iei/saved_images DISPLAY_IMG=flase
```
Now there will no longer by a video output - only the JSON object in the terminal:

```json
{"Measurement":"stream1_results","Channels":3.0,"Height":1080.0,"ImgHandle":"persist_2687d37d","Width":1920.0,"cam_sn":"camera-serial-number","defects":"[]","display_info":"[{\"info\":\"Inferencetime:24.679ms\",\"priority\":0},{\"info\":\"Throughput:40.521fps\",\"priority\":0},{\"info\":\"WorkerSafe:True\",\"priority\":0}]","encoding":"jpg","idx":65649.0,"image_id":"18a6d4bd-d446-4b88-8edc-983eab4d99ef","machine_id":"tool_2","part_id":"376101d0-268d-4c30-98a4-78725fe8847d","timestamp":1567197537.8904457,"influx_ts":1567197537891778821
```

If you check the "Throughput" it should increase substantially over the pervious run. 

### Offloading Workload 

Another way to increase performance and free up CPU capacity is by running the same code on various Intel's Accelerators such as GPUs and Intel® Myriad™-VPUs.

The Restricted zone notifier application can be run on different hardwares by customizing the configuration JSON file (restricted_zone_notifier.json).

### Run application with GPU
- To Run on GPU we need to edit the restricted_zone_notifier.json file.
Execute the following commands:

    ```bash
    cd ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/config/alog_config/
    sudo gedit restricted_zone_notifier.json
    ```
- Change the device to GPU ```device=GPU``` inside JSON file

Now re run the application:

```bash
    cd ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup
    sudo make provision CERT_PATH=../cert-tool/Certificates/
    sudo make install CERT_PATH=../cert-tool/Certificates/
```
- Next check to see if the EIS pipeline is running properly: 

```bash
    tail -f /opt/intel/iei/logs/consolidatedLogs/iei.log
```
Now launch the visualizer: 

```bash
cd ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer
    sudo make build
    sudo make run CERT_PATH=~/Workshop/IEdgeInsights-v1.5LTS/cert-tool/Certificates/ HOST=localhost IMAGE_DIR=/opt/intel/iei/saved_images DISPLAY_IMG=true
```
This will lower the stress on the CPU and cepending on the CPU and GPU of your device you will see a performance increase. 

### Next, inference with Myriad™ VPU
The Myriad™ Inference Engine plugin supports VPU devices such as the Intel® Neural Compute Stick.

- To run on an Intel® Myriad VPU, change from ```"device":GPU``` to ```"device":MYRIAD```.

- VPU devices only support FP16 data type. So we need to use the FP16 variant of our pre-trained person detection model. The pre-trained models are available in the following path.
    ```
    ~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/config/alog_config/restricted_zone_notifier  
    ```
- To complete this steps, replace the following lines of code:

    ```JSON
    "model_xml": "./algos/algo_config/restricted_zone_notifier/person-detection-retail-0013.xml",
    "model_bin": "./algos/algo_config/restricted_zone_notifier/person-detection-retail-0013.bin",
    "device": "GPU"
    ```

- with below lines of codes in **restricted_zone_notifier.json** file.

    ```JSON
    "model_xml": "./algos/algo_config/restricted_zone_notifier/person-detection-retail-0013-fp16.xml",
    "model_bin": "./algos/algo_config/restricted_zone_notifier/person-detection-retail-0013-fp16.bin",
    "device": "MYRIAD"
    ```
Now re run the application:

```bash
    cd ~/IEdgeinsights/docker_setup/
    sudo make provision CERT_PATH=../cert-tool/Certificates/
    sudo make install CERT_PATH=../cert-tool/Certificates/
```

### Lesson Learnt

We have created a custom application that has the user deploy the Restricted Zone Notifier Reference Implementation using the Edge Insights Software framework.

[Home](./README.md)

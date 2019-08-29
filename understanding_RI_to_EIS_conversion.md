# How to Convert a python based RI to Classifier and Trigger based Intel® Edge Insights (EIS) Software

In this lab, We will understand how to port an application developed in python and run as single script to the classifier and trigger based Intel® Edge Insights (EIS) Software framework. We will start with the **Restricted Zone Notifier** reference implementation which can be found here: https://github.com/intel-iot-devkit/restricted-zone-notifier-python.

## Understanding Restricted Zone Notifier code flow
This sample is intended to demonstrate how to use Inference Engine included in the Intel® Distribution of OpVIenNO™ toolkit and the Intel® Deep Learning Deployment Toolkit can be used to improve assembly line safety for human operators and factory workers.

First, let us have a look on the below code flow for the restricted zone notifier application.

![](images/flowchart.jpg)

**Now**, Lets understand how we can convert this python based RI to EIS framework step by step.

### 1. Provide input to run the application

In Python based reference implementation the model to use in classification, plugin libraries, the hardware to run the code on, and the source video are set on the command line at launch. In the EIS framework, a JSON file is created that has all of these parameters set. The JSON file is available in the **IEdgeInsights/docker_setup/config/algo_config** directory.
![](images/rzn_input.png)


### 2. Initialization and loading IR to the plugin for target device

  In the EIS framework, a **Classifier** module is provided where the plugin initialization and inference can be done.

  The Classifier module has two methods: `__init__` and `classify`.

  `__init__`  : This method is used for initialization and loading the intermediate representation model to the plugin for the target device.  
  `classify` : This method is used for inferencing and capturing the inference output.
  
To create the __init__.py module the initialization parameters from the main() function of our pytthon script are ported over. 
  
![](images/rzn_initialization.png)

### 3. Capture frames from input file
- In the python based code, the  frames are captured from image/video using cv2.Videocapture.


- In EIS framework, the fframe capture is done using the **Trigger** module.The trigger module in the EIS stack receives the captured frames from the VideoIngestion module. The purpose of the trigger algorithm is to select frames of interest from the camera stream and pass them to the VideoAnalytics module. The algorithm to use in the trigger modul depends on the use case. For people monitoring you might need a classifier algorithm to execute on all frames whereas a use case like the sample application “pcbdemo” would need the classifier to execute on only the frames where the PCB board is in the center of the frame.

- After the video frames are processed by the trigger they are registered by the register_trigger_callback() method to be passed to the classifier. 

The section of code in main.py that deals with the video frame input is ported to the resttricted zone notifier trigger script.

![](images/rzn_trigger.png)

### 4. Execute SSD model and parse results

This part of the sub-module needs to be done just after the inferencing results. So it can be included in the classifier `__init__.py` is available in IEdgeInsights/algos/dpm/classification

The porting has to be done as below:
![](images/rzn_ssd_out.png)

### 5. Update Status and Alert information
- The status and alert information is also included in classifier `__init__.py` is available in IEdgeInsights/algos/dpm/classification/classifiers directory.
classify() returns display and defect information.

![](images/rzn_output.png)

### 6. Messaging Thread
In Python based RI Worker thread (Messaging Thread) that publishes MQTT messages to Server to display the output.

In EIS framework, a OPC/UA based visualizer application is available. We will use that to view the resulting output.
The warning message from the classifier will be displayed using visualizer application along with inference time details. So the display and defect information will be send to OPC/UA server from classifier.

Now we understood, how a python based RI can be converted to Classifier and Trigger based Intel® Edge Insights (EIS) Software. So, let's implement in our next lab.

## Next Lab
[Deploying Restricted Zone Notifier using Edge Insights Software framework](./lab_restricted_zone_notifier.md)

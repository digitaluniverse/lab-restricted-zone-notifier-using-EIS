# How to Convert a python based RI to Classifier and Trigger based Intel® Edge Insights (EIS) Software

In this lab, We will understand how to port an application developed in python and run as single script to the classifier and trigger based Intel® Edge Insights (EIS) Software framework. We will start with the **Restricted Zone Notifier** reference implementation which can be found here: https://github.com/intel-iot-devkit/restricted-zone-notifier-python.

## Understanding Restricted Zone Notifier code flow
This sample is intended to demonstrate how to use Inference Engine included in the Intel® Distribution of OpenVINO™ toolkit and the Intel® Deep Learning Deployment Toolkit can be used to improve assembly line safety for human operators and factory workers.

First, let us have a look on the below code flow for the restricted zone notifier application.

![](images/flowchart.jpg)

**Now**, Lets understand how we can convert this python based RI to EIS framework step by step.

### 1. Provide input to run the application

In Python based reference implementation the model to use in classification, plugin libraries, the hardware to run the code on, and the source video are set on the command line at launch. In the EIS framework, a JSON file is created that has all of these parameters set. The JSON file is available in the **IEdgeInsights/docker_setup/config/algo_config** directory.
![](images/rzn_input_1.png)
![](images/arrow.png)
![](images/rzn_input_2.png)


### 2. Initialization and loading IR to the plugin for target device

  In the EIS framework, a **Classifier** module is provided where the plugin initialization and inference can be done.

  The Classifier module has two methods: `__init__` and `classify`.

  `__init__`  : This method is used for initialization and loading the intermediate representation model to the plugin for the target device.  
  `classify` : This method is used for inferencing and capturing the inference output.
  
To create the __init__.py module the initialization parameters from the main() function of our pytthon script are ported over. 
  
![](images/rzn_initialization_1.png)
![](images/arrow.png)
![](images/rzn_initialization_2.png)

### 3. Capture frames from input file
- In the python based code, the  frames are captured from image/video using cv2.Videocapture.


- In EIS framework, the fframe capture is done using the **Trigger** module.The trigger module in the EIS stack receives the captured frames from the VideoIngestion module. The purpose of the trigger algorithm is to select frames of interest from the camera stream and pass them to the VideoAnalytics module. The algorithm to use in the trigger modul depends on the use case. For people monitoring you might need a classifier algorithm to execute on all frames whereas a use case like the sample application “pcbdemo” would need the classifier to execute on only the frames where the PCB board is in the center of the frame.

- After the video frames are processed by the trigger they are registered by the register_trigger_callback() method to be passed to the classifier. 

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

## Explore Intel® Edge Insights Software
### What is Intel® Edge Insights Software
Industrial Edge Insights Software (EIS) from Intel is a scalable solution for the industrial sector addressing various manufacturing uses which include data collection, storage, and analytics on a variety of hardware nodes that span across the factory floor.
### How Industrial Edge Insights Software Works
To understand how it works, let's first understand key components of this software stack.

**Configuration File (factory_pcbdemo.json)**   
This JSON file is the main configuration file for the entire data pipeline. Using this file, a user can define the data ingestion, storage, triggers, and classifiers to be used in the application.

*Location:*~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/config/algo_config/factory_pcbdemo.json

Take a look at the file now and notice some key elements: 

Video Ingestion:

```json
"data_ingestion_manager": {
        "ingestors": {
            "video_file": {
                "video_file": "./test_videos/pcb_d2000.avi",
                "encoding": {
                    "type": "jpg",
                    "level": 100
                },
                "img_store_type": "inmemory_persistent",
                "loop_video": true,
                "poll_interval": 0.2
            }
        }
    }
```

Here we see the video file set as "./test_videos/pcb_d2000.avi" this path is relative to ~/Workshop/IEdgeInsights-V1.5LTS/docker_setup

Trigger Setup:

```json
    "triggers": {
        "pcb_trigger": {
            "training_mode": false,
            "n_total_px": 300000,
            "n_left_px": 1000,
            "n_right_px": 1000
        }
```

Here we see the trigger set as "pcb_trigger" - this specifices that we will use ~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/triggers/pcb_trigger.py as the trigger script for our application. The script that creates the trigger is ~/Workshop/IEdgeInsights-v1.5LTS/VideoIngestion/VideoIngestion.py 


Classifier Setup:
```json
 "classification": {
        "max_workers": 1,
        "classifiers": {
            "pcbdemo": {
                "trigger": [
                    "pcb_trigger"
                ],
                "config": {
                    "ref_img": "./algos/algo_config/ref_pcbdemo/ref.png",
                    "ref_config_roi": "./algos/algo_config/ref_pcbdemo/roi_2.json",
                    "model_xml": "./algos/algo_config/ref_pcbdemo/model_2.xml",
                    "model_bin": "./algos/algo_config/ref_pcbdemo/model_2.bin",
                    "device": "CPU"
                }
            }
        }
    }
```

This block defines the classification module where we specify the classifer name "pcbdemo" - This will be used by ~/Workshop/IEdgeInsights-V1.5LTS/algos/dpm/classification/classifier_manager.py to set the connection between the Trigger module and Classification module by using the ~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/classification/classifiers/pcbdemo folder as the location of the classification scripts.

Note: This folder must be in this location and have the same name as the classifier to function. 

This block also sets the reference image and regioin of interest files as well as the intermediate representation model files for our inference model. 

**Video Ingestion**   
The Video Ingestion module in the EIS is a user defined function, which uses the Data Ingestion library to ingest data to InfluxDB and Image store

![](images/VideoIngestion.png)

*Location:*~/Workshop/IEdgeInsights-v1.5LTS/VideoIngestion/

**Trigger**   
This filters the incoming data stream, mainly to reduce the storage and computation requirements by only passing on frames of interest. All input frames are passed to the Trigger.  When it detects a frame of interest based on user defined functions, it activates a signal which causes that frame to be saved in the Image Store database, and the metadata for that frame in the InfluxDB database.

*Location:*~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/triggers/

**Classifier**   
The is a user defined algorithm that is run on each frame of interest. Kapacitor, an open source data processing engine, subscribes to the meta-data stream , and the classifier receives the meta-data from Kapacitor. The classifier pulls the frame from the Image Store, and saves the analysis results as metadata back into the InfluxDB database.

*Location:*~/Workshop/IEdgeInsights-v1.5LTS/algos/dpm/classification/classifiers/

**visualizer**   
      This is to view resulting output

*Location:*~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer/


***By now we understood the components, Let's understand how data flows between these components.***

![](images/eis-overview.png)

**Step-1.a:**
The Video Ingestion module starts capturing frames from Basler camera /
RTSP camera (or video file) and sends the frames to the Trigger Algorithm.

**Step-1.b:**
The Trigger Algorithm will determine the relevant frames that are to go to
the Classifier. In the PCB demo use case, the Trigger Algorithm selects the images with the full PCB within view and send relavant frames to teh Video Ingestion.

**Step-2:**
The relevant frames from the Trigger Algorithm are stored into Image Store and the corresponding meta-data is stored in InfluxDB

**Step-3:**
Kapacitor subscribes to the Meta Data stream. All the streams in InfluxDB are subscribed as default by Kapacitor.

**Step-4:**
The Classifier UDF (User Defined Function) receives the Meta Data from Kapacitor. It then invokes the UDF classifier algorithm.

**Step-5:**
UDF Classifier Algorithm fetches the image frame from the Image Store and
generates the Classified Results with defect information, if any. Each frame
produces one set of Classified Results.

**Step-6:**
The Classifier UDF returns the classified results to Kapacitor.

**Step-7:**
The Kapacitor saves the Classified Results to InfluxDB. This behavior is enforced in the Kapacitor TICK script.

**Step-8:**
The Classified Results are received by Factory Control Application.

**Step-9:**
The Factory Control Application will turn on the alarm light if a defect is found in the part.

**Step-10:**
The Stream Manager subscribes to the Classified Results stream. The policy of stream export is set in the Stream Manager.

**Step-11:**
The Stream Manager uses the Data Bus Abstraction module interfaces to publish the Classified Results stream. The Data Bus Abstraction provides a publish-subscribe interface.

**Step-12:**
Data Bus Abstraction creates an OPC-UA Server, which exposes the Classified Results data as a string.

**Step-13:**
The Classified result is published in the OPC-UA message bus and available to external applications

**Step-14:**
The image handle of the actual frame is part of the Classified Results. The raw image can be retrieved through the data agent using the GetBlob() API.

**Step-15:**
The raw image frame is returned in response to the GetBlob() command.


**NOTE:** As an application developer, you do not need to worry about handling the data flow described above from data ingestion to classification. The included software stack using InfluxDB and Kapacitor handle the data movement and storage.



### Running Defect detection demo application                            
**Description**   
Printed Circuits Boards(PCBs) are being inspected for quality control to check if any defects(missing component or components are short) are there with the PCBs. To find out the defects a good quality PCB will be compared against the defective ones and pin point the location of the defect as well.

Input to the application can be from a live stream or from a video file. Here video file (~/Workshop/IEdgeInsights-v1.5LTS/docker_setup/test_videos/pcb_d2000.avi) is used for this demo.

**Build and Run Sample.**  
To **build and run** the sample pcbdemo sample application execute the following commands.

```bash
cd ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer/
sudo make build run
```

Once this completes run the following command to view the log:

```bash
tail -f /opt/intel/iei/logs/consolidatedLogs/iei.log
```
If everything is running properly you will see:

```
ia_data_agent         | I0829 11:46:55.835922       6 StreamManager.go:191] Publishing topic: stream1_results
```
Which is indicating that the ia_data_agent container is streaming data on the "stream1_results" topic. 

To **Visualize** the sample application , execute the below command:

```bash
cd ~/Workshop/IEdgeInsights-v1.5LTS/tools/visualizer
source ./source.sh
python3 visualize.py -D true
```
This will run the vizualizer in developer mode (no certificates needed). 

**Pcb-Demo Output**   
Once the application successfully runs. The output window will be poped up as below.
![](images/pcbdemo_result.png)


You should now understand the Intel® Edge Insights Software framework components and how run pcbdemo application successfully.    
Let's Deploy a Restricted Zone Notifier Reference implementation using Intel® Edge Insights Software framework in our next lab.

## Next Lab
[Understanding of Converting Python based RI to classifier and trigger based IEI Software](./understanding_ri_to_eis_conversion.md)

# Azure Open AI o1-preview model - Root cause analysis for manufacturing line Troubleshooting

## Introduction

- Goal is to show How to find root cause on machine failures in factory floor.
- Based on equipment ontology, alarms and event and historical sensor data.
- Idea is to reduce downtime and increase efficiency.
- Provide current constraints and future constraints.
- Given the equipment ontology can be multi hierarchical, the model can be used to find the root cause of the issue.
- Help find issues and fix faster for better efficiency.

## Pre-requisites

- Azure subscription
- Azure Open AI o1-preview model
- Local computer
- Visual Studio Code
- python 3.12 with virtual environment
- Make sure to store the endpoing and key in the .env file
- Model version to use: 2024-08-01-preview

## Steps

### Environment Setup

- create a .env file

```
AZURE_OPENAI_O1_ENDPOINT="https://api.openai.com"
AZURE_OPENAI_O1_KEY="yourkey"
```

### Code

- Create a new python file
- install necessary libraries
- Create .env file and setup the necessary keys
- import the libraries

```
import pandas as pd
import os

from pprint import pprint
from azure.ai.evaluation import evaluate
from openai import AzureOpenAI

from dotenv import load_dotenv
```

- Load the environment variables

```
load_dotenv()
```

- Initialize the Azure Open AI client

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_O1_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_O1_KEY"),  
  api_version="2024-08-01-preview",
)
```

- Here is the function to process the supply chain sourcing
- i am including the data in the prompt as hardcoded sample dataset
- for real world application, add your own data and adjust the prompt

```
def processo1text(query):
    returntxt = ""

    rfttext = ""

    message_text = [
    {"role":"user", "content":f"""I am providing 3 datasets one for meta data, one for alarms and event, one for historical data with sensor. Your job is to analyze the latest fault and find the root cause from which equipment and subsystem it originated.

    Here is the meta data:
    Machine_ID,Machine_Name,Location,Sensors_Associated  
    M001,Conveyor Belt,Section A,"Sensor_1, Sensor_2"  
    M002,Heat Exchanger,Section B,"Sensor_3, Sensor_4"  
    M003,Compressor,Section C,"Sensor_5, Sensor_6"  
    M004,Hydraulic Press,Section D,"Sensor_7, Sensor_8"  
    M005,Packaging Machine,Section E,"Sensor_9, Sensor_10"  
    M006,Assembly Robot,Section F,"Sensor_11, Sensor_12"  
    M007,Paint Booth,Section G,"Sensor_13, Sensor_14"  
    M008,CNC Machine,Section H,"Sensor_15, Sensor_16"  
    M009,Lathe Machine,Section I,"Sensor_17, Sensor_18"  
    M010,Welding Station,Section J,"Sensor_19, Sensor_20"  
    M011,Drill Press,Section K,"Sensor_21, Sensor_22"  
    M012,Injection Molder,Section L,"Sensor_23, Sensor_24"  
    M013,Grinder,Section M,"Sensor_25, Sensor_26"  
    M014,Shearing Machine,Section N,"Sensor_27, Sensor_28"  
    M015,Water Jet Cutter,Section O,"Sensor_29, Sensor_30"  
    M016,Laser Cutter,Section P,"Sensor_31, Sensor_32"  
    M017,3D Printer,Section Q,"Sensor_33, Sensor_34"  
    M018,Printing Press,Section R,"Sensor_35, Sensor_36"  
    M019,Textile Loom,Section S,"Sensor_37, Sensor_38"  
    M020,Blender,Section T,"Sensor_39, Sensor_40"  
    M021,Extruder,Section U,"Sensor_41, Sensor_42"  
    M022,Granulator,Section V,"Sensor_43, Sensor_44"  
    M023,Boiler,Section W,"Sensor_45, Sensor_46"  
    M024,Reactor,Section X,"Sensor_47, Sensor_48"  
    M025,Filter,Section Y,"Sensor_49, Sensor_50"  


    Here is the alarms and event data:

    Timestamp,Event_ID,Event_Description,Severity,Related_Sensor  
    2023-01-01 02:00,E001,Overheat Detected,High,Sensor_3  
    2023-01-01 04:00,E002,Low Pressure Alert,Medium,Sensor_5  
    2023-01-01 06:00,E003,Vibration Alert,High,Sensor_7  
    2023-01-01 08:00,E004,High Temperature,Critical,Sensor_9  
    2023-01-01 10:00,E005,Power Surge Detected,High,Sensor_10  
    2023-01-01 12:00,E006,Overheat Detected,High,Sensor_3  
    2023-01-01 14:00,E007,Low Pressure Alert,Medium,Sensor_5  
    2023-01-01 16:00,E008,Vibration Alert,High,Sensor_7  
    2023-01-01 18:00,E009,High Temperature,Critical,Sensor_9  
    2023-01-01 20:00,E010,Power Surge Detected,High,Sensor_10  
    2023-01-01 22:00,E011,Overheat Detected,High,Sensor_3  
    2023-01-02 00:00,E012,Low Pressure Alert,Medium,Sensor_5  
    2023-01-02 02:00,E013,Vibration Alert,High,Sensor_7  
    2023-01-02 04:00,E014,High Temperature,Critical,Sensor_9  
    2023-01-02 06:00,E015,Power Surge Detected,High,Sensor_10  
    2023-01-02 08:00,E016,Overheat Detected,High,Sensor_3  
    2023-01-02 10:00,E017,Low Pressure Alert,Medium,Sensor_5  
    2023-01-02 12:00,E018,Vibration Alert,High,Sensor_7  
    2023-01-02 14:00,E019,High Temperature,Critical,Sensor_9  
    2023-01-02 16:00,E020,Power Surge Detected,High,Sensor_10  
    2023-01-02 18:00,E021,Overheat Detected,High,Sensor_3  
    2023-01-02 20:00,E022,Low Pressure Alert,Medium,Sensor_5  
    2023-01-02 22:00,E023,Vibration Alert,High,Sensor_7  
    2023-01-03 00:00,E024,High Temperature,Critical,Sensor_9  
    2023-01-03 02:00,E025,Power Surge Detected,High,Sensor_10  
    2023-01-03 04:00,E026,Overheat Detected,High,Sensor_3  
    2023-01-03 06:00,E027,Low Pressure Alert,Medium,Sensor_5  
    2023-01-03 08:00,E028,Vibration Alert,High,Sensor_7  
    2023-01-03 10:00,E029,High Temperature,Critical,Sensor_9  
    2023-01-03 12:00,E030,Power Surge Detected,High,Sensor_10  
    2023-01-03 14:00,E031,Overheat Detected,High,Sensor_3  
    2023-01-03 16:00,E032,Low Pressure Alert,Medium,Sensor_5  
    2023-01-03 18:00,E033,Vibration Alert,High,Sensor_7  
    2023-01-03 20:00,E034,High Temperature,Critical,Sensor_9  
    2023-01-03 22:00,E035,Power Surge Detected,High,Sensor_10  
    2023-01-04 00:00,E036,Overheat Detected,High,Sensor_3  
    2023-01-04 02:00,E037,Low Pressure Alert,Medium,Sensor_5  
    2023-01-04 04:00,E038,Vibration Alert,High,Sensor_7  
    2023-01-04 06:00,E039,High Temperature,Critical,Sensor_9  
    2023-01-04 08:00,E040,Power Surge Detected,High,Sensor_10  
    2023-01-04 10:00,E041,Overheat Detected,High,Sensor_3  
    2023-01-04 12:00,E042,Low Pressure Alert,Medium,Sensor_5  
    2023-01-04 14:00,E043,Vibration Alert,High,Sensor_7  
    2023-01-04 16:00,E044,High Temperature,Critical,Sensor_9  
    2023-01-04 18:00,E045,Power Surge Detected,High,Sensor_10  
    2023-01-04 20:00,E046,Overheat Detected,High,Sensor_3  
    2023-01-04 22:00,E047,Low Pressure Alert,Medium,Sensor_5  
    2023-01-05 00:00,E048,Vibration Alert,High,Sensor_7  
    2023-01-05 02:00,E049,High Temperature,Critical,Sensor_9  
    2023-01-05 04:00,E050,Power Surge Detected,High,Sensor_10  


    Here is the historical data:
    Timestamp,Sensor_1,Sensor_2,Sensor_3,Sensor_4,Sensor_5,Sensor_6,Sensor_7,Sensor_8,Sensor_9,Sensor_10  
    2023-01-01 00:00,0.50,1.20,0.90,0.70,1.10,0.80,0.60,1.00,0.40,0.80  
    2023-01-01 01:00,0.55,1.15,0.95,0.75,1.15,0.85,0.65,1.05,0.45,0.85  
    2023-01-01 02:00,0.60,1.10,1.00,0.80,1.20,0.90,0.70,1.10,0.50,0.90  
    2023-01-01 03:00,0.65,1.05,1.05,0.85,1.25,0.95,0.75,1.15,0.55,0.95  
    2023-01-01 04:00,0.70,1.00,1.10,0.90,1.30,1.00,0.80,1.20,0.60,1.00  
    2023-01-01 05:00,0.75,0.95,1.15,0.95,1.35,1.05,0.85,1.25,0.65,1.05  
    2023-01-01 06:00,0.80,0.90,1.20,1.00,1.40,1.10,0.90,1.30,0.70,1.10  
    2023-01-01 07:00,0.85,0.85,1.25,1.05,1.45,1.15,0.95,1.35,0.75,1.15  
    2023-01-01 08:00,0.90,0.80,1.30,1.10,1.50,1.20,1.00,1.40,0.80,1.20  
    2023-01-01 09:00,0.95,0.75,1.35,1.15,1.55,1.25,1.05,1.45,0.85,1.25  
    2023-01-01 10:00,1.00,0.70,1.40,1.20,1.60,1.30,1.10,1.50,0.90,1.30  
    2023-01-01 11:00,1.05,0.65,1.45,1.25,1.65,1.35,1.15,1.55,0.95,1.35  
    2023-01-01 12:00,1.10,0.60,1.50,1.30,1.70,1.40,1.20,1.60,1.00,1.40  
    2023-01-01 13:00,1.15,0.55,1.55,1.35,1.75,1.45,1.25,1.65,1.05,1.45  
    2023-01-01 14:00,1.20,0.50,1.60,1.40,1.80,1.50,1.30,1.70,1.10,1.50  
    2023-01-01 15:00,1.25,0.45,1.65,1.45,1.85,1.55,1.35,1.75,1.15,1.55  
    2023-01-01 16:00,1.30,0.40,1.70,1.50,1.90,1.60,1.40,1.80,1.20,1.60  
    2023-01-01 17:00,1.35,0.35,1.75,1.55,1.95,1.65,1.45,1.85,1.25,1.65  
    2023-01-01 18:00,1.40,0.30,1.80,1.60,2.00,1.70,1.50,1.90,1.30,1.70  
    2023-01-01 19:00,1.45,0.25,1.85,1.65,2.05,1.75,1.55,1.95,1.35,1.75  
    2023-01-01 20:00,1.50,0.20,1.90,1.70,2.10,1.80,1.60,2.00,1.40,1.80  
    2023-01-01 21:00,1.55,0.15,1.95,1.75,2.15,1.85,1.65,2.05,1.45,1.85  
    2023-01-01 22:00,1.60,0.10,2.00,1.80,2.20,1.90,1.70,2.10,1.50,1.90  
    2023-01-01 23:00,1.65,0.05,2.05,1.85,2.25,1.95,1.75,2.15,1.55,1.95  
    2023-01-02 00:00,1.70,0.00,2.10,1.90,2.30,2.00,1.80,2.20,1.60,2.00  
    2023-01-02 01:00,1.75,0.05,2.15,1.95,2.35,2.05,1.85,2.25,1.65,2.05  
    2023-01-02 02:00,1.80,0.10,2.20,2.00,2.40,2.10,1.90,2.30,1.70,2.10  
    2023-01-02 03:00,1.85,0.15,2.25,2.05,2.45,2.15,1.95,2.35,1.75,2.15  
    2023-01-02 04:00,1.90,0.20,2.30,2.10,2.50,2.20,2.00,2.40,1.80,2.20  
    2023-01-02 05:00,1.95,0.25,2.35,2.15,2.55,2.25,2.05,2.45,1.85,2.25  
    2023-01-02 06:00,2.00,0.30,2.40,2.20,2.60,2.30,2.10,2.50,1.90,2.30  
    2023-01-02 07:00,2.05,0.35,2.45,2.25,2.65,2.35,2.15,2.55,1.95,2.35  
    2023-01-02 08:00,2.10,0.40,2.50,2.30,2.70,2.40,2.20,2.60,2.00,2.40  
    2023-01-02 09:00,2.15,0.45,2.55,2.35,2.75,2.45,2.25,2.65,2.05,2.45  
    2023-01-02 10:00,2.20,0.50,2.60,2.40,2.80,2.50,2.30,2.70,2.10,2.50  
    2023-01-02 11:00,2.25,0.55,2.65,2.45,2.85,2.55,2.35,2.75,2.15,2.55  
    2023-01-02 12:00,2.30,0.60,2.70,2.50,2.90,2.60,2.40,2.80,2.20,2.60  
    2023-01-02 13:00,2.35,0.65,2.75,2.55,2.95,2.65,2.45,2.85,2.25,2.65  
    2023-01-02 14:00,2.40,0.70,2.80,2.60,3.00,2.70,2.50,2.90,2.30,2.70  
    2023-01-02 15:00,2.45,0.75,2.85,2.65,3.05,2.75,2.55,2.95,2.35,2.75  
    2023-01-02 16:00,2.50,0.80,2.90,2.70,3.10,2.80,2.60,3.00,2.40,2.80  
    2023-01-02 17:00,2.55,0.85,2.95,2.75,3.15,2.85,2.65,3.05,2.45,2.85  
    2023-01-02 18:00,2.60,0.90,3.00,2.80,3.20,2.90,2.70,3.10,2.50,2.90  
    2023-01-02 19:00,2.65,0.95,3.05,2.85,3.25,2.95,2.75,3.15,2.55,2.95  
    2023-01-02 20:00,2.70,1.00,3.10,2.90,3.30,3.00,2.80,3.20,2.60,3.00  
    2023-01-02 21:00,2.75,1.05,3.15,2.95,3.35,3.05,2.85,3.25,2.65,3.05  
    2023-01-02 22:00,2.80,1.10,3.20,3.00,3.40,3.10,2.90,3.30,2.70,3.10  
    2023-01-02 23:00,2.85,1.15,3.25,3.05,3.45,3.15,2.95,3.35,2.75,3.15  
    2023-01-03 00:00,2.90,1.20,3.30,3.10,3.50,3.20,3.00,3.40,2.80,3.20  
    2023-01-03 01:00,2.95,1.25,3.35,3.15,3.55,3.25,3.05,3.45,2.85,3.25  
    2023-01-03 02:00,3.00,1.30,3.40,3.20,3.60,3.30,3.10,3.50,2.90,3.30  
    2023-01-03 03:00,3.05,1.35,3.45,3.25,3.65,3.35,3.15,3.55,2.95,3.35  
    2023-01-03 04:00,3.10,1.40,3.50,3.30,3.70,3.40,3.20,3.60,3.00,3.40  
    2023-01-03 05:00,3.15,1.45,3.55,3.35,3.75,3.45,3.25,3.65,3.05,3.45  
    2023-01-03 06:00,3.20,1.50,3.60,3.40,3.80,3.50,3.30,3.70,3.10,3.50  
    2023-01-03 07:00,3.25,1.55,3.65,3.45,3.85,3.55,3.35,3.75,3.15,3.55  
    2023-01-03 08:00,3.30,1.60,3.70,3.50,3.90,3.60,3.40,3.80,3.20,3.60  
    2023-01-03 09:00,3.35,1.65,3.75,3.55,3.95,3.65,3.45,3.85,3.25,3.65  
    2023-01-03 10:00,3.40,1.70,3.80,3.60,4.00,3.70,3.50,3.90,3.30,3.70  
    2023-01-03 11:00,3.45,1.75,3.85,3.65,4.05,3.75,3.55,3.95,3.35,3.75  
    2023-01-03 12:00,3.50,1.80,3.90,3.70,4.10,3.80,3.60,4.00,3.40,3.80  
    2023-01-03 13:00,3.55,1.85,3.95,3.75,4.15,3.85,3.65,4.05,3.45,3.85  
    2023-01-03 14:00,3.60,1.90,4.00,3.80,4.20,3.90,3.70,4.10,3.50,3.90  
    2023-01-03 15:00,3.65,1.95,4.05,3.85,4.25,3.95,3.75,4.15,3.55,3.95  
    2023-01-03 16:00,3.70,2.00,4.10,3.90,4.30,4.00,3.80,4.20,3.60,4.00  
    2023-01-03 17:00,3.75,2.05,4.15,3.95,4.35,4.05,3.85,4.25,3.65,4.05  
    2023-01-03 18:00,3.80,2.10,4.20,4.00,4.40,4.10,3.90,4.30,3.70,4.10  
    2023-01-03 19:00,3.85,2.15,4.25,4.05,4.45,4.15,3.95,4.35,3.75,4.15  
    2023-01-03 20:00,3.90,2.20,4.30,4.10,4.50,4.20,4.00,4.40,3.80,4.20  
    2023-01-03 21:00,3.95,2.25,4.35,4.15,4.55,4.25,4.05,4.45,3.85,4.25  
    2023-01-03 22:00,4.00,2.30,4.40,4.20,4.60,4.30,4.10,4.50,3.90,4.30  
    2023-01-03 23:00,4.05,2.35,4.45,4.25,4.65,4.35,4.15,4.55,3.95,4.35  

     if the question is outside the bounds of the RFP, Let the user know answer might be relevant for data provided.
     If not sure, ask the user to provide more information.
     {query}"""},]

    response = client.chat.completions.create(
        model= "o1-preview", #"gpt-4-turbo", # model = "deployment_name".
        messages=message_text,
    )

    returntxt = response.choices[0].message.content
    return returntxt
```

- Call the function with the query

```
if __name__ == "__main__":
    query = "Create a detailed Root cause analysis and find the issue and also see if you get resolution and then create a report to submit?"
    print(processo1text(query))
```

- now run the o1test.py file

```
python o1test.py
```

- wait for the output

## output

```
---

# Root Cause Analysis Report

## 1. Introduction

This report presents a detailed root cause analysis of the latest fault occurring within the facility's equipment. Using the provided datasets—metadata of machines and sensors, alarms and events logs, and historical sensor data—we aim to identify the origin of the fault, determine the underlying cause, and recommend corrective actions.

## 2. Summary of Available Data

### 2.1 Metadata of Machines and Sensors

- **Machine ID:** M005
- **Machine Name:** Packaging Machine
- **Location:** Section E
- **Sensors Associated:** Sensor_9, Sensor_10

### 2.2 Alarms and Events Data

The latest events recorded are:

| Timestamp           | Event ID | Event Description       | Severity | Related Sensor |
|---------------------|----------|-------------------------|----------|----------------|
| 2023-01-05 02:00    | E049     | High Temperature        | Critical | Sensor_9       |
| **2023-01-05 04:00** | **E050** | **Power Surge Detected** | **High**  | **Sensor_10**   |

### 2.3 Historical Sensor Data (up to 2023-01-03 23:00)

The historical data includes time-stamped readings from Sensor_1 to Sensor_10. Notably, Sensor_10 shows a consistent upward trend in its readings over time.

---

## 3. Latest Fault Identification

### 3.1 Fault Details

- **Timestamp:** **2023-01-05 04:00**
- **Event ID:** **E050**
- **Event Description:** **Power Surge Detected**
- **Severity:** **High**
- **Related Sensor:** **Sensor_10**

### 3.2 Affected Equipment

- **Machine ID:** **M005**
- **Machine Name:** **Packaging Machine**
- **Location:** **Section E**
- **Subsystem:** **Electrical System**
- **Sensors Monitoring the Equipment:** Sensor_9 (Temperature), Sensor_10 (Electrical Parameters)

---

## 4. Analysis of Historical Data

### 4.1 Sensor_10 Trends

- **Observation Period:** 2023-01-01 00:00 to 2023-01-03 23:00
- **Trend:** Steady increase in Sensor_10 readings from **0.80** to **4.35**
- **Interpretation:** The rising values may indicate escalating electrical load or deteriorating electrical components leading to a potential power surge.

### 4.2 Correlation with Events

- **Repeated Events:** "Power Surge Detected" events associated with Sensor_10 occur every 48 hours.
- **Pattern:** The consistent timing suggests a cyclical issue possibly linked to operational schedules or periodic processes affecting the electrical system.

---

## 5. Root Cause Determination

### 5.1 Potential Causes

- **Electrical Overload:** The increasing Sensor_10 readings suggest that the Packaging Machine's electrical system is experiencing higher than normal loads.
- **Component Degradation:** Wear and tear of electrical components (e.g., capacitors, wiring insulation) could lead to inefficiencies and surges.
- **External Factors:** Fluctuations in the facility's power supply or harmonics in the electrical network affecting sensitive equipment.

### 5.2 Supporting Evidence

- **Consistent Increase in Readings:** Sensor_10 data shows a clear upward trajectory without stabilization.
- **Recurrent High-Severity Events:** Multiple high-severity alarms related to electrical anomalies suggest an unresolved underlying issue.
- **Proximity to Other Faults:** Prior critical event at 2023-01-05 02:00 (E049 - High Temperature) involving Sensor_9, which is also on the Packaging Machine, indicating thermal stress possibly affecting electrical performance.

---

## 6. Limitations

- **Incomplete Data:** Historical sensor data is missing from 2023-01-04 00:00 onwards, limiting the ability to analyze conditions immediately preceding the latest fault.
- **Lack of Maintenance Records:** No data on recent maintenance activities which could influence equipment performance.
- **Absence of External Factors Data:** No information on power supply quality or environmental conditions that may impact the equipment.

---

## 7. Recommendations

### 7.1 Immediate Actions

1. **Conduct an Electrical Inspection:**
   - Check for signs of overheating, burning smells, or discolored components in the Packaging Machine's electrical panel.
   - Test voltage levels and electrical connections for irregularities.

2. **Monitor Temperature Levels:**
   - Assess Sensor_9 readings for overheating, given the prior high-temperature alarm.
   - Inspect cooling systems or ventilation affecting the Packaging Machine.

3. **Isolate the Equipment:**
   - If feasible, temporarily shut down the Packaging Machine to prevent further damage and ensure safety.

### 7.2 Preventive Measures

1. **Install Power Quality Equipment:**
   - Use surge protectors or uninterruptible power supplies (UPS) to safeguard against external power fluctuations.

2. **Schedule Regular Maintenance:**
   - Implement a maintenance plan focusing on electrical components prone to wear.

3. **Update Monitoring Systems:**
   - Ensure real-time data collection from all sensors to prevent future data gaps.
   - Set up automated alerts when sensor readings exceed predefined thresholds.

### 7.3 Further Investigation

1. **Data Collection:**
   - Retrieve missing historical data post-2023-01-03 23:00 for comprehensive analysis.
   - Collect environmental data (e.g., ambient temperature, humidity) that could affect equipment performance.

2. **Cross-Departmental Review:**
   - Collaborate with the facility's electrical engineers and maintenance team to identify any recent changes or anomalies in power supply.

3. **Assess Systemic Issues:**
   - Review events from other machines experiencing similar issues to determine if there's a broader systemic problem.

---

## 8. Conclusion

The latest fault, a **Power Surge Detected** event (E050) on **2023-01-05 04:00**, originated from the **Packaging Machine (M005)** in **Section E**, specifically linked to **Sensor_10** monitoring electrical parameters. The root cause is likely an electrical overload or component degradation within the machine's electrical subsystem.

Consistent increases in Sensor_10 readings and repeated high-severity alarms support this conclusion. However, missing historical data limits a full assessment. Immediate inspection and corrective actions are recommended to prevent equipment damage and operational disruptions.

---

## 9. Appendices

### 9.1 Event Timeline for Sensor_10

| Timestamp           | Event ID | Event Description       | Severity | Related Sensor |
|---------------------|----------|-------------------------|----------|----------------|
| 2023-01-01 10:00    | E005     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-01 20:00    | E010     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-02 06:00    | E015     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-02 16:00    | E020     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-03 02:00    | E025     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-03 12:00    | E030     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-03 22:00    | E035     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-04 08:00    | E040     | Power Surge Detected    | High     | Sensor_10      |
| 2023-01-04 18:00    | E045     | Power Surge Detected    | High     | Sensor_10      |
| **2023-01-05 04:00** | **E050** | **Power Surge Detected** | **High**  | **Sensor_10**   |

### 9.2 Historical Sensor_10 Readings (Selected)

| Timestamp           | Sensor_10 Reading |
|---------------------|-------------------|
| 2023-01-01 00:00    | 0.80              |
| 2023-01-01 12:00    | 1.40              |
| 2023-01-02 00:00    | 2.00              |
| 2023-01-02 12:00    | 2.60              |
| 2023-01-03 00:00    | 3.20              |
| 2023-01-03 12:00    | 3.80              |
| 2023-01-03 23:00    | 4.35              |

---

## 10. Closing Remarks

Addressing the identified electrical issues promptly is crucial to maintaining operational efficiency and safety within the facility. Implementing the recommended actions will help mitigate risks and prevent future occurrences of similar faults.

---
```


## conclusion
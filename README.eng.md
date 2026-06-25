# 🚦 AI-Based Smart Traffic Management System

<p>
  <img src="https://img.shields.io/badge/LANGUAGE-Python-3776AB?style=for-the-badge&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/AI%20MODEL-YOLOv8-00FFFF?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/VISION-OpenCV-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white"/>
  <img src="https://img.shields.io/badge/INFERENCE-ONNX-005CED?style=for-the-badge&logo=onnx&logoColor=white"/>
</p>

<p>
  <img src="https://img.shields.io/badge/EDGE%20DEVICE-Raspberry%20Pi%205-A22846?style=for-the-badge&logo=raspberrypi&logoColor=white"/>
  <img src="https://img.shields.io/badge/MCU-Arduino%20MEGA%202560-00979D?style=for-the-badge&logo=arduino&logoColor=white"/>
  <img src="https://img.shields.io/badge/DATABASE-SQLite-003B57?style=for-the-badge&logo=sqlite&logoColor=white"/>
  <img src="https://img.shields.io/badge/DATASET-Roboflow-6706CE?style=for-the-badge"/>
</p>

This project is an AI-based smart traffic management system built on a Raspberry Pi 5.
It uses a YOLO object detection model and OpenCV-based image processing to analyze real-time traffic scenes and automatically detect traffic violations or hazardous events.

The system determines the position and movement of pedestrians and vehicles based on predefined Regions of Interest (ROIs), such as crosswalks, roads, illegal parking zones, and U-turn areas.
When an event occurs, the system captures images, stores event logs, and manages statistics using an SQLite database.

---

## 📌 Project Overview

### Project Period

2026.03.06 ~ 2026.03.21

* Won an Encouragement Award at the DeviceMart 2026 ICT Convergence Project Contest

### Project Goals

* Detect traffic violations and hazardous situations using real-time video analysis
* Recognize pedestrians, vehicles, emergency vehicles, and fall events using a YOLO model
* Analyze traffic situations through OpenCV-based ROI processing
* Save event images and record logs in an SQLite database
* Control external alert devices by integrating Raspberry Pi with Arduino
* Build a system that can operate in both real road-like scenarios and a scaled demonstration environment

---

## 🛠 Technologies and Hardware

| Category         | Technology / Device      | Description                                                                       |
| ---------------- | ------------------------ | --------------------------------------------------------------------------------- |
| Programming      | Python                   | Implemented the main control system and event decision logic                      |
| AI Model         | YOLOv8                   | Detected pedestrians, vehicles, emergency vehicles, and fall events               |
| Image Processing | OpenCV                   | Processed video frames, defined ROIs, analyzed object positions, and saved images |
| Edge Device      | Raspberry Pi 5           | Ran real-time video analysis and the overall system                               |
| Camera           | Intel RealSense D435i    | Used as the main video input device and for distance information                  |
| Camera           | PILOMAX USB Webcam       | Used as a secondary camera for blind spots and event image capture                |
| Database         | SQLite                   | Stored event logs, image paths, timestamps, and statistical data                  |
| Display          | XPT2046 Touch Controller | Displayed the Raspberry Pi-based UI screen                                        |
| External Control | Arduino MEGA 2560 R3     | Controlled LEDs, NeoPixels, and external alert devices                            |
| Communication    | Serial Communication     | Transmitted status commands between Raspberry Pi and Arduino                      |
| Labeling Tool    | AnyLabeling              | Labeled custom data collected from the scaled model environment                   |
| Dataset Platform | Roboflow                 | Used for initial object detection model training data                             |

---

## 🧠 System Workflow

<p align="center">
  <img src="./imgs/full_structure.png" width="850"><br>
  <b>Smart Traffic Management System Workflow</b>
</p>

1. Receive real-time video input through cameras
2. Detect pedestrians, vehicles, emergency vehicles, and fall events using a YOLO model
3. Analyze the overlap between object bounding boxes and OpenCV-based ROI areas
4. Determine traffic events based on signal status, object position, movement direction, and duration
5. Save event images and record logs in the SQLite database when an event occurs
6. Send commands to Arduino when needed to control LEDs or NeoPixel alerts
7. Display the current system status and event results on the screen

---

## 📋 Main Features

### 1. Crosswalk Jaywalking Detection

When the pedestrian signal is red and a pedestrian enters the crosswalk ROI beyond a predefined threshold, the system detects it as a jaywalking event.

* Calculates the overlap ratio between the pedestrian object and the crosswalk ROI
* Determines the event based on the pedestrian signal status
* Saves an image when an event occurs
* Records the timestamp, event type, and image path in the SQLite database
* Applies a cooldown period to reduce repeated detections of the same object
* Turns on the red NeoPixel LED during nighttime conditions

---

### 2. Road Jaywalking Detection

When a pedestrian enters a road ROI outside the crosswalk area, the system detects it as a dangerous jaywalking event regardless of the traffic signal status.

* Checks whether the pedestrian object overlaps with the road ROI
* Detects road intrusion outside the crosswalk area
* Captures the jaywalking scene using the secondary USB webcam
* Saves event images and logs
* Turns on the red NeoPixel LED during nighttime conditions

---

### 3. Vehicle Crosswalk Violation Detection

When a vehicle enters the crosswalk area while the vehicle signal is red, the system detects it as a vehicle crosswalk violation.

* Calculates the overlap ratio between the vehicle object and the crosswalk ROI
* Determines the event based on the vehicle signal status
* Turns on the blue NeoPixel LED during nighttime conditions
* Records the event on the display and in the database

---

### 4. Illegal Parking Detection

When a vehicle remains inside the illegal parking ROI for a certain period of time, the system detects it as an illegal parking event.

* Measures how long the vehicle stays inside the designated parking violation area
* Triggers an event when the duration exceeds the threshold
* Saves the vehicle image
* Records the timestamp and image path in the database

---

### 5. Illegal U-Turn Detection

When a vehicle sequentially enters specific road ROIs or moves across both lane areas, the system detects it as an illegal U-turn event.

* Analyzes vehicle position changes and ROI entry conditions
* Checks the overlap ratio across both road areas
* Saves an image of the illegal U-turn vehicle
* Records the event log in the database

---

### 6. Fall Detection

When the YOLO model detects a fall class, the system handles it as an emergency event rather than a general traffic violation.

* Triggers an emergency event when the fall class is detected
* Displays the emergency situation on the screen
* Handles the event with higher priority than general traffic violations

---

### 7. Emergency Vehicle Detection

When an emergency vehicle or an active emergency light is detected, the system switches to an emergency traffic control state.

* Detects emergency vehicle-related classes
* Maintains the emergency state for a certain duration
* Turns on all red, yellow, and green vehicle traffic lights
* Displays an emergency alert message on the screen

---

### 8. Event Logging and Statistics Management

All detected events are stored in an SQLite database for later review and analysis.

The stored information includes:

* Event timestamp
* Event type
* Saved image path
* Daily accumulated event count
* Statistical data for analysis

---

## ⚙️ Data Training and Preprocessing

### 1. Initial Training with Public Datasets

<p align="center">
  <img src="./imgs/real_model_test.png" width="750"><br>
  <b>Initial Model Test Result Using Public Datasets</b>
</p>

For the initial model training, public pedestrian and vehicle datasets from Roboflow were used.
This provided a baseline object detection performance, which was then improved through additional training for the project-specific environment.

### 2. Custom Data Collection in the Scaled Model Environment

<p align="center">
  <img src="./imgs/labeling_test.png" width="750"><br>
  <b>Labeling Process for the Scaled Model Environment</b>
</p>

<p align="center">
  <img src="./imgs/labeling_result.jpg" width="750"><br>
  <b>Labeling Result of the Custom Model Data</b>
</p>

Since the demonstration environment used small model vehicles and pedestrian objects, the initial dataset was not sufficient to achieve reliable detection performance.

To solve this issue, custom images were collected directly from the scaled model environment.
The objects were labeled using AnyLabeling, and the dataset was used for additional model training.

### 3. Model Optimization

The model was optimized to run in real time on a Raspberry Pi 5 while maintaining a balance between detection accuracy and inference speed.

* Tested the initial model
* Added custom training data from the scaled model environment
* Retrained the model based on YOLOv8
* Used an ONNX model considering Raspberry Pi inference speed
* Optimized frame processing for simultaneous use of two cameras

---

## 📷 System Architecture

<p align="center">
  <img src="./imgs/system_architecture.png" width="850"><br>
  <b>Smart Traffic Management System Architecture</b>
</p>

### Hardware Components

* Raspberry Pi 5
* Intel RealSense Depth Camera D435i
* PILOMAX USB Webcam
* Arduino MEGA 2560 R3
* XPT2046 Touch Controller
* NeoPixel LED
* Traffic light model
* Road and crosswalk model

### Software Components

* Python-based main control program
* YOLOv8 object detection model
* OpenCV image processing
* SQLite database
* Arduino LED control code
* Serial communication-based device control

---

## 🖥️ Demo Screens

### Crosswalk Jaywalking Detection

| Real Demo                                                    | Display Demo                                                    |
| ------------------------------------------------------------ | --------------------------------------------------------------- |
| <img src="./gifs/real_crosswalk_jaywalking.gif" width="420"> | <img src="./gifs/display_crosswalk_jaywalking.gif" width="420"> |

---

* A pedestrian enters the crosswalk ROI while the vehicle traffic light is red
* The system detects jaywalking when the ROI overlap ratio exceeds the threshold
* The secondary webcam displays the jaywalking area
* Event image and timestamp are saved in the database
* The red NeoPixel LED turns on during nighttime conditions

### Road Jaywalking Detection

| Real Demo                                               | Display Demo                                               |
| ------------------------------------------------------- | ---------------------------------------------------------- |
| <img src="./gifs/real_road_jaywalking.gif" width="420"> | <img src="./gifs/display_road_jaywalking.gif" width="420"> |

---

* A pedestrian moves into the road area instead of the crosswalk
* An event is triggered when the pedestrian intrudes into the road ROI regardless of signal status
* Event image and log are saved
* The red LED indicates the hazardous situation at night

### Illegal Parking Detection

| Real Demo                                               | Display Demo                                               |
| ------------------------------------------------------- | ---------------------------------------------------------- |
| <img src="./gifs/real_illegal_parking.gif" width="420"> | <img src="./gifs/display_illegal_parking.gif" width="420"> |

---

* A vehicle remains in the illegal parking zone for a predefined period
* An event is triggered when the stopped duration exceeds the threshold
* The vehicle image is saved
* The event timestamp and image path are recorded in the database

### Illegal U-Turn Detection

| Real Demo                                             | Display Demo                                             |
| ----------------------------------------------------- | -------------------------------------------------------- |
| <img src="./gifs/real_illegal_uturn.gif" width="420"> | <img src="./gifs/display_illegal_uturn.gif" width="420"> |

---

* A vehicle enters the U-turn restricted area or crosses both road ROIs
* The system determines an illegal U-turn based on vehicle position changes and ROI conditions
* Event image and log are saved

### Vehicle Crosswalk Violation Detection

| Real Demo                                                           | Display Demo                                                           |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| <img src="./gifs/real_vehicle_crosswalk_violation.gif" width="420"> | <img src="./gifs/display_vehicle_crosswalk_violation.gif" width="420"> |

---

* A vehicle enters the crosswalk ROI while the vehicle signal is red
* A vehicle crosswalk violation event is triggered
* The blue NeoPixel LED turns on during nighttime conditions

### Fall Detection

| Real Demo                                              | Display Demo                                              |
| ------------------------------------------------------ | --------------------------------------------------------- |
| <img src="./gifs/real_fall_detection.gif" width="420"> | <img src="./gifs/display_fall_detection.gif" width="420"> |

---

* An emergency event is triggered when the fall class is detected
* The event is handled separately from general traffic violations
* The emergency situation is displayed on the screen

### Emergency Vehicle Detection

| Real Demo                                                          | Display Demo                                                          |
| ------------------------------------------------------------------ | --------------------------------------------------------------------- |
| <img src="./gifs/real_emergency_vehicle_yielding.gif" width="420"> | <img src="./gifs/display_emergency_vehicle_yielding.gif" width="420"> |

---

* An emergency vehicle or active emergency light is detected
* All three vehicle traffic lights are turned on to indicate the emergency state
* An emergency message is displayed on the screen

---

## 💻 Display and Nighttime NeoPixel LED

<div>
  <img src="imgs/real_display.jpg" height="400">
  <img src="imgs/red_LED.png" style="width:250px; height:400px">
  <img src="imgs/blue_LED.png" style="width:250px; height:400px">
</div>

### 1. Display: XPT2046 Touch Controller

* Connected to Raspberry Pi 5 to display the system UI screen

### 2. Nighttime Jaywalking Alert

* Turns on the red NeoPixel LED when a pedestrian crosses during a red pedestrian signal or enters the road outside the crosswalk area

### 3. Nighttime Vehicle Violation Alert

* Turns on the blue NeoPixel LED when a vehicle enters the crosswalk while the pedestrian signal is green

---

## 💾 Database Management

When an event occurs, the following information is stored in the SQLite database.

### 1. Database Tables

<img src="imgs/db_table1.png" height="200">
<img src="imgs/db_table2.png" height="200">

### 2. Stored Event Images

<img src="imgs/db_image1.png" height="200">
<img src="imgs/db_image2.png" height="200">

| Field      | Description                           |
| ---------- | ------------------------------------- |
| id         | Unique event ID                       |
| event_type | Type of detected event                |
| timestamp  | Time when the event occurred          |
| image_path | Path of the saved event image         |
| date       | Date information for daily statistics |

The database was used not only to detect events in real time, but also to accumulate event history for later review and analysis.

---

## ⚠️ Troubleshooting

### 🚦 Traffic Light Misclassified as a Person

<p>
<img src="imgs/troubleshooting1.png" width="300"/>
<img src="imgs/troubleshooting2.png" width="300"/>
</p>

* **Problem:** After training the model with red pedestrian signal data, the red traffic light was sometimes misclassified as a `person` class.
* **Solution:** Person detections inside a specific traffic light ROI were ignored to prevent false positive detections.

### 🚗 Vehicle Class Not Detected at Night

<p>
<img src="imgs/troubleshooting3.png" width="300"/>
<img src="imgs/troubleshooting4.png" width="300"/>
</p>

* **Problem:** When daytime and nighttime vehicles were trained as a single `vehicle` class, the model failed to detect vehicles at night.
* **Solution:** The vehicle class was separated into daytime and nighttime classes, such as `vehicle` and `carnight`, to improve nighttime detection performance.

### 🚶 Pedestrian Detection Failure

<p>
<img src="imgs/troubleshooting5.png" width="800"/>
</p>

* **Problem:** YOLOv5n was used to maintain camera processing speed while using two cameras, but it failed to detect pedestrians reliably.
* **Solution:** The model was retrained with YOLOv8s to improve detection accuracy, and the trained `best.pt` model was converted to `best.onnx` to improve inference speed.

---

## 📈 Future Improvements

* More detailed pedestrian classification
  Example: stroller, wheelchair, walking aid

* Detection of additional road hazards
  Example: fallen objects, obstacles, trash, timers, animals

* Integration with autonomous driving and smart vehicles
  Example: V2X communication

---

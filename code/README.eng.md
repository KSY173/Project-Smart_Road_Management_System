## 📄 best.onnx File

---

### [best.onnx](https://drive.google.com/file/d/1XqK4mp-QP13xLfzmXe0lj93abNRFLfwH/view?usp=sharing)

### 📦 Model Info

- Format: ONNX
- Model: YOLOv8s
- Input Size: 320 x 320
- Channels: 3 (RGB)
- Batch Size: 1
- Task: Object Detection

### 🎯 Classes

- vehicle
- person
- fall
- ON
- OFF
- carnight
- personnight
- fallnight

### ⚡ Features

- Lightweight model optimized for the Raspberry Pi environment
- Supports real-time object detection
- Custom-trained model based on YOLOv8s

---

## 🧠 Core Feature Description

---

### 1️⃣ System Initialization and Integrated Control Structure

- **Core Code**

```python
# =========================
# System Initialization
# =========================

# Initialize DB
init_db()

# Load YOLO model
model = YOLO(MODEL_PATH)

# Connect Arduino serial communication
ser = serial.Serial(SERIAL_PORT, BAUDRATE, timeout=1)

# Initialize auxiliary camera (USB webcam)
aux_cap = open_aux_camera(AUX_CAM_INDEX)

# Initialize RealSense D435
pipeline = rs.pipeline()
config = rs.config()
config.enable_stream(rs.stream.color, 640, 480, rs.format.bgr8, 30)
config.enable_stream(rs.stream.depth, 640, 480, rs.format.z16, 30)
```

- **Description**
  - Initializes the YOLO ONNX model, Intel RealSense D435, USB webcam, SQLite database, and Arduino serial communication.

---

### 2️⃣ ROI-Based Area Configuration

- **Core Code**

```python
# =========================
# Fixed ROI for D435
# =========================

LEFT_ROAD_ROIS = [
    (229, 136, 348, 286),
    (206, 170, 231, 290),
    (177, 209, 203, 288),
    (152, 240, 176, 290),
    (1, 429, 635, 477),
]

# Right-lane areas
RIGHT_ROAD_ROIS = [
    (355, 132, 500, 280),
    (499, 154, 514, 280),
    (514, 182, 534, 279),
    (532, 211, 551, 277),
]

# Entire road area
ROAD_ROIS = LEFT_ROAD_ROIS + RIGHT_ROAD_ROIS

# Crosswalk areas
CROSSWALK_ROIS = [
    (63, 294, 635, 427),
    (17, 353, 61, 426),
]

# Fixed coordinates of the traffic signal area
SIGNAL_ROI = (568, 111, 628, 133)
```

- **Description**
  - Road areas, crosswalk areas, and the traffic signal area were manually defined using ROI (Region of Interest) coordinates.
  - The system then calculates the overlap ratio between the detected pedestrian or vehicle objects and the ROIs to determine location-based events such as jaywalking, vehicle intrusion, and illegal U-turns.

---

### 3️⃣ SQLite-Based Event Storage Structure

- **Core Code**

```python
# =========================
# SQLite-Based Event Storage Structure
# =========================

def init_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Create event log table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS event_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            event_type TEXT NOT NULL,
            detected_at TEXT NOT NULL,
            image_path TEXT
        )
    """)

    # Create daily jaywalking statistics table
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS daily_stats (
            stat_date TEXT PRIMARY KEY,
            jaywalk_count INTEGER NOT NULL DEFAULT 0,
            last_detect_time TEXT
        )
    """)

    conn.commit()
    conn.close()


def save_event(event_type, detected_at, image_path=None):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    cursor.execute("""
        INSERT INTO event_logs (event_type, detected_at, image_path)
        VALUES (?, ?, ?)
    """, (event_type, detected_at, image_path))

    conn.commit()
    conn.close()


def update_jaywalk_daily_stats(detected_at):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Extract date (YYYY-MM-DD)
    date = detected_at[:10]

    cursor.execute("""
        SELECT jaywalk_count FROM daily_stats WHERE stat_date = ?
    """, (date,))
    row = cursor.fetchone()

    if row:
        # If data already exists, increment the count
        cursor.execute("""
            UPDATE daily_stats
            SET jaywalk_count = jaywalk_count + 1,
                last_detect_time = ?
            WHERE stat_date = ?
        """, (detected_at, date))
    else:
        # If no data exists, create a new record
        cursor.execute("""
            INSERT INTO daily_stats (stat_date, jaywalk_count, last_detect_time)
            VALUES (?, 1, ?)
        """, (date, detected_at))

    conn.commit()
    conn.close()
```

- **Description**
  - An SQLite database structure was designed to store event logs and daily statistics.
  - The `event_logs` table stores the event type, detection time, and image path.
  - The `daily_stats` table manages the number of jaywalking events per day and the latest detection time.
  - This structure enables not only real-time event detection, but also post-event log review and statistical analysis.
  - When an event occurs, the database stores the event information together with the saved image path so that it can be linked to an event image viewer later.

---

### 4️⃣ OpenCV-Based Traffic Light and Warning Light State Detection

- **Core Code**

```python
# =========================
# OpenCV-Based Traffic Light / Warning Light State Detection
# =========================

def judge_warning_light(roi):
    if roi is None or roi.size == 0:
        return "OFF", 0.0, 0.0, None        # If the ROI is empty, treat it as OFF

    hsv = cv2.cvtColor(roi, cv2.COLOR_BGR2HSV)  # Convert RGB to HSV, which is better for separating red/green color ranges

    lower_red_1 = np.array([0, 80, 80])         # Red range 1
    upper_red_1 = np.array([10, 255, 255])
    lower_red_2 = np.array([160, 80, 80])       # Red range 2
    upper_red_2 = np.array([180, 255, 255])

    mask1 = cv2.inRange(hsv, lower_red_1, upper_red_1)      # Red mask 1
    mask2 = cv2.inRange(hsv, lower_red_2, upper_red_2)      # Red mask 2
    red_mask = cv2.bitwise_or(mask1, mask2)                 # Merge the two masks

    kernel = np.ones((3, 3), np.uint8)          # Kernel for noise removal
    red_mask = cv2.morphologyEx(red_mask, cv2.MORPH_OPEN, kernel)   # Remove small noise
    red_mask = cv2.morphologyEx(red_mask, cv2.MORPH_DILATE, kernel) # Expand the red area

    red_pixels = cv2.countNonZero(red_mask)         # Number of red pixels
    total_pixels = roi.shape[0] * roi.shape[1]      # Total number of ROI pixels
    red_ratio = red_pixels / total_pixels if total_pixels > 0 else 0.0  # Red pixel ratio

    v_channel = hsv[:, :, 2]    # Brightness channel (V)
    mean_v = cv2.mean(v_channel, mask=red_mask)[0] if red_pixels > 0 else 0.0   # Average brightness of the red area

    RED_RATIO_THRESHOLD = 0.03  # Red ratio of at least 3%
    BRIGHTNESS_THRESHOLD = 180  # Brightness check to verify that the warning light is actually on rather than simply detecting a red object

    state = "ON" if red_ratio >= RED_RATIO_THRESHOLD and mean_v >= BRIGHTNESS_THRESHOLD else "OFF"
    return state, red_ratio, mean_v, red_mask



# Traffic signal color judgment function
# Since the signal position is fixed, the judgment is performed using fixed coordinates
def judge_traffic_signal(roi):
    if roi is None or roi.size == 0:
        return "UNKNOWN", 0.0, 0.0, None, None

    hsv = cv2.cvtColor(roi, cv2.COLOR_BGR2HSV)

    lower_red_1 = np.array([0, 80, 80])
    upper_red_1 = np.array([10, 255, 255])
    lower_red_2 = np.array([160, 80, 80])
    upper_red_2 = np.array([180, 255, 255])

    lower_green = np.array([40, 80, 80])        # Green range
    upper_green = np.array([90, 255, 255])

    red_mask1 = cv2.inRange(hsv, lower_red_1, upper_red_1)
    red_mask2 = cv2.inRange(hsv, lower_red_2, upper_red_2)
    red_mask = cv2.bitwise_or(red_mask1, red_mask2)
    green_mask = cv2.inRange(hsv, lower_green, upper_green) # Green mask

    kernel = np.ones((3, 3), np.uint8)
    red_mask = cv2.morphologyEx(red_mask, cv2.MORPH_OPEN, kernel)
    red_mask = cv2.morphologyEx(red_mask, cv2.MORPH_DILATE, kernel)
    green_mask = cv2.morphologyEx(green_mask, cv2.MORPH_OPEN, kernel)
    green_mask = cv2.morphologyEx(green_mask, cv2.MORPH_DILATE, kernel)

    total_pixels = roi.shape[0] * roi.shape[1]
    red_pixels = cv2.countNonZero(red_mask)
    green_pixels = cv2.countNonZero(green_mask)

    red_ratio = red_pixels / total_pixels if total_pixels > 0 else 0.0
    green_ratio = green_pixels / total_pixels if total_pixels > 0 else 0.0

    RED_THRESHOLD = 0.03
    GREEN_THRESHOLD = 0.03

    if red_ratio >= RED_THRESHOLD and red_ratio > green_ratio:          # If the red ratio is large enough and greater than green, classify as RED
        signal_state = "RED"
    elif green_ratio >= GREEN_THRESHOLD and green_ratio > red_ratio:    # If the green ratio is large enough and greater than red, classify as GREEN
        signal_state = "GREEN"
    else:                                                               # If both are ambiguous, classify as UNKNOWN
        signal_state = "UNKNOWN"

    return signal_state, red_ratio, green_ratio, red_mask, green_mask
```

- **Description**
  - The actual ON/OFF state of traffic lights and warning lights was determined using OpenCV HSV color-space conversion and mask operations.
  - Instead of relying only on YOLO object detection results, the system additionally analyzes the color ratio inside the ROI to distinguish red lights, green lights, and the emergency warning light ON state.
  - This enables state-based event judgment beyond simple object detection.

---

### 5️⃣ Jaywalking Decision Logic

- **Core Code**

```python
# =========================
# Jaywalking Decision Logic
# =========================

# Core jaywalking logic
                elif label_name_lower in {c.lower() for c in PERSON_CLASSES}:
                    person_road_ratio_max = 0.0
                    # Calculate the maximum overlap between the pedestrian box and road ROIs
                    for road_roi in road_rois:
                        ratio = overlap_ratio(det_box, road_roi)
                        person_road_ratio_max = max(person_road_ratio_max, ratio)

                    person_cross_ratio_max = 0.0
                    # Calculate the maximum overlap between the pedestrian box and crosswalk ROIs
                    for cw in crosswalk_rois:
                        person_cross_ratio_max = max(person_cross_ratio_max, overlap_ratio(det_box, cw))

                    # Detect jaywalking when a pedestrian enters at least 1/3 of the crosswalk area while the vehicle signal is green,
                    # or when the pedestrian enters at least 1/3 of the road area instead of using the crosswalk, regardless of the signal state
                    is_jaywalking = (person_road_ratio_max >= (1 / 3)) or ((signal_state == "GREEN") and (person_cross_ratio_max >= (1 / 3)))

                    # Draw a red bounding box when jaywalking is detected
                    if is_jaywalking:
                        jaywalking_detected = True
                        color = (0, 0, 255)
                        text = f"{label_name} JAYWALK"
                    else:
                        color = (0, 255, 255)
                        text = f"{label_name}"

                    cv2.rectangle(display_frame, (x1, y1), (x2, y2), color, 2)
```

- **Description**
  - When a pedestrian is detected in the crosswalk while the vehicle signal is green, the system calculates the overlap ratio with the crosswalk ROI.
  - When a pedestrian enters a road area outside the crosswalk, the system calculates the overlap ratio with the road ROI regardless of the signal state.
  - If the overlap ratio exceeds the predefined threshold, the situation is classified as jaywalking.
  - This design allows the system to interpret real traffic violations based on the object’s position instead of simply detecting a person.

---

### 6️⃣ Distance Measurement Using the RealSense D435

- **Core Code**

```python
# =========================
# Distance Measurement Using D435
# =========================

# Distance calculation
                    if is_jaywalking:
                        # Calculate the center point of the pedestrian
                        cx = int((x1 + x2) / 2)
                        cy = int((y1 + y2) / 2)
                        cv2.circle(display_frame, (cx, cy), 5, (0, 0, 255), -1) # Mark the center point

                        person_depth = get_valid_depth_median(depth_frame, cx, cy, patch_size=PATCH_SIZE) # Get the depth value at the pedestrian center point


                        if person_depth > 0: # Validate the depth value
                            prepare_line_points() # Prepare the reference line

                            # Convert the pedestrian center point to 3D coordinates
                            if line_points_3d_cache:
                                person_3d = deproject_to_3d(intr, cx, cy, person_depth)

                                min_dist_m = float("inf")
                                closest_line_pixel = None

                                for lpix, l3d in zip(line_pixels_cache, line_points_3d_cache): # Find the minimum distance by comparing the pedestrian with each sampled reference-line point
                                    dist = euclidean_distance(person_3d, l3d)

                                    # Find the smallest distance between the pedestrian and the sampled points on the reference line
                                    if dist < min_dist_m:
                                        min_dist_m = dist
                                        closest_line_pixel = lpix

                                dist_cm = min_dist_m * 100.0    # Convert the D435 distance value from meters to centimeters
                                current_min_dist_cm = dist_cm
```

- **Description**
  - The system uses depth information from the Intel RealSense D435 to calculate the real-world distance to the center point of a detected pedestrian.
  - After obtaining the depth value at the center point, the 2D pixel coordinate is converted into a 3D spatial coordinate.
  - The Euclidean distance between the pedestrian point and sampled points on the crosswalk reference line is then calculated to obtain the minimum distance.
  - This allows the system to display jaywalking situations with actual distance information in centimeters, rather than only detecting the event.

---

### 7️⃣ Vehicle Event Decision Logic

- **Core Code**

```python
# =========================
# Vehicle Intrusion Event Decision
# =========================

# Vehicle class
                elif label_name_lower in {c.lower() for c in VEHICLE_CLASSES}:
                    left_ratio = overlap_ratio_with_mask(det_box, left_road_mask)       # How much the vehicle overlaps with the left lane
                    right_ratio = overlap_ratio_with_mask(det_box, right_road_mask)     # How much the vehicle overlaps with the right lane
                    illegal_zone_ratio = overlap_ratio_with_mask(det_box, illegal_zone_mask)  # How much the vehicle overlaps with the illegal zone

                    cross_ratio_max = 0.0

                     # Calculate the degree of crosswalk intrusion
                    for cw in crosswalk_rois:
                        cross_ratio_max = max(cross_ratio_max, overlap_ratio(det_box, cw))

                    # Detect vehicle intrusion when the vehicle enters at least 1/3 of the crosswalk ROI while the vehicle signal is red
                    intrusion_cond = (signal_state == "RED") and (cross_ratio_max >= (1 / 3))
                    if intrusion_cond:
                        vehicle_intrusion_detected = True
    illegal_zone_ratio = 0.0
    for roi in CROSSWALK_ROIS:
        illegal_zone_ratio = max(illegal_zone_ratio, overlap_ratio(vehicle_box, roi))

# =========================
# Illegal U-Turn Event Decision
# =========================
   # Detect an illegal U-turn when a vehicle simultaneously overlaps at least 1/3 of both the left and right lanes
                    illegal_uturn_cond = (left_ratio >= ROAD_RATIO_THRESHOLD) and (right_ratio >= ROAD_RATIO_THRESHOLD)
                    if illegal_uturn_cond:
                        illegal_uturn_detected = True
                        if score > best_uturn_score:
                            best_uturn_score = score
                            best_uturn_box = det_box

# =========================
# Illegal Parking Event Decision
# =========================
                   # Calculate illegal parking
                    if illegal_zone_ratio >= ILLEGAL_ZONE_RATIO_THRESHOLD:
                        if vehicle_key not in illegal_park_state:   # If this is a newly detected vehicle, record the start time
                            illegal_park_state[vehicle_key] = {
                                "start_time": now,
                                "reported": False
                            }   
                        else:   # If the vehicle has already been detected, update last_seen
                            parked_time = now - illegal_park_state[vehicle_key]["start_time"]
                            if parked_time >= ILLEGAL_PARK_SEC: # Confirm illegal parking if the vehicle remains in the illegal parking zone for at least 3 seconds
                                illegal_park_state[vehicle_key]["reported"] = True
                                illegal_parking_detected = True
                                if illegal_zone_ratio > best_illegal_parking_ratio:
                                    best_illegal_parking_ratio = illegal_zone_ratio
                                    best_illegal_parking_box = det_box
```

- **Description**
  - When a vehicle is detected while the vehicle signal is red, the system calculates the overlap ratio between the vehicle and the crosswalk ROI.
  - When a vehicle object is detected, the system also calculates the overlap ratios with the left-lane and right-lane ROIs.
  - Based on these values, the system classifies vehicle intrusion, illegal U-turns, and illegal parking.
  - In particular, illegal parking is defined as a case where a vehicle remains in a specific illegal zone for a certain amount of time. This enables time-based event judgment beyond simple location-based detection.

---

### 8️⃣ Fall Detection and Emergency-State Hold Logic

- **Core Code**

```python
# =========================
# Fall Detection
# =========================

# Fall class
                elif label_name_lower in {c.lower() for c in FALL_CLASSES}:
                    signal_overlap = overlap_ratio(det_box, signal_roi)

                    fall_roi_overlap_max = 0.0
                    for roi in fall_valid_rois:
                        fall_roi_overlap_max = max(fall_roi_overlap_max, overlap_ratio(det_box, roi))

                    is_valid_fall = (signal_overlap < 0.30) and (fall_roi_overlap_max >= FALL_VALID_RATIO_THRESHOLD)

                    if is_valid_fall:
                        fall_detected = True
                        cv2.rectangle(display_frame, (x1, y1), (x2, y2), (255, 0, 255), 2)
                        cv2.putText(display_frame, f"{label_name} {score:.2f}",
                                    (x1, max(y1 - 10, 20)),
                                    cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255, 0, 255), 2)
                        cv2.putText(display_frame, f"fall_roi={fall_roi_overlap_max:.2f}",
                                    (x1, min(y2 + 18, h - 10)),
                                    cv2.FONT_HERSHEY_SIMPLEX, 0.4, (255, 0, 255), 1)
                    else:
                        cv2.rectangle(display_frame, (x1, y1), (x2, y2), (120, 120, 120), 1)
                        cv2.putText(display_frame, f"IGNORE {label_name} {score:.2f}",
                                    (x1, max(y1 - 10, 20)),
                                    cv2.FONT_HERSHEY_SIMPLEX, 0.5, (180, 180, 180), 1)
                        cv2.putText(display_frame, f"fall_roi={fall_roi_overlap_max:.2f}",
                                    (x1, min(y2 + 18, h - 10)),
                                    cv2.FONT_HERSHEY_SIMPLEX, 0.4, (180, 180, 180), 1)

                else:
                    color = (0, 255, 255)
                    cv2.rectangle(display_frame, (x1, y1), (x2, y2), color, 2)
                    cv2.putText(display_frame, f"{label_name} {score:.2f}",
                                (x1, max(y1 - 10, 20)),
                                cv2.FONT_HERSHEY_SIMPLEX, 0.7, color, 2)

# =========================
# Emergency-State Hold
# =========================
            raw_on_detected = emergency_detected
            raw_fall_detected = fall_detected

            # Start the 15-second timer only when the warning light is newly detected as ON,
            # so the timer does not extend indefinitely while it remains on
            if raw_on_detected and (not prev_raw_on_detected):
                ambulance_emergency_until = now + AMBULANCE_EMERGENCY_DURATION
                last_detect_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

            fall_emergency_active = raw_fall_detected
            ambulance_emergency_active = (now < ambulance_emergency_until)
            emergency_now = fall_emergency_active or ambulance_emergency_active

            if emergency_now:
                emergency_status = "DETECTED"
                show_emergency_fullscreen = True
                show_capture_fullscreen = False
            else:
                emergency_status = "NOT DETECTED"
                show_emergency_fullscreen = False

            if show_capture_fullscreen and now >= show_capture_until:
                show_capture_fullscreen = False

# =========================
# Turn On All Three Vehicle Signals When the Warning Light Is Detected as ON
# =========================
warning_override_active = (now - hold_state["last_warning_on_time"]) <= WARNING_HOLD_SECONDS # Maintain the emergency state if it is within 3 seconds of the most recent ON detection
    hold_state["signal_cmd"] = "TL_ALL" if warning_override_active else "TL_NORMAL" # Use TL_ALL while the warning light state is being held; otherwise use TL_NORMAL

    status_text = "USB EMERGENCY LIGHT ON" if emergency_detected_now else "USB NORMAL"
    status_color = (0, 0, 255) if emergency_detected_now else (0, 255, 0)
    cv2.putText(out, status_text, (15, 35), cv2.FONT_HERSHEY_SIMPLEX, 0.8, status_color, 2)
```

- **Description**
  - The system determines an emergency state when the `fall` class is detected or when the emergency warning light is detected as ON.
  - When the warning light is detected as ON, the system switches to the emergency state and turns on all three vehicle traffic lights so that the emergency situation can be clearly recognized.

---

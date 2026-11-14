# TKR 2026 Image-Processing Codes

Computer-vision and navigation prototype for the TKR 2026 Robocon robot. The
program combines three camera feeds, two custom Ultralytics YOLO models, HSV
colour segmentation, and serial commands from a controller to:

1. detect and align with the zone-1 spear/palm/fist target;
2. classify red objects in a 3-column, 4-row arena section; and
3. select the robot's next grid position while avoiding symbol/fake targets.


## System overview

```text
Spherical camera (index 3) ──> spf3.pt + HSV red/green masks ──> Zone 1 alignment

Left camera (index 1)  ─┐
                        ├─> symbol300.pt ─> Cam_data.py ─> Row_data.py
Right camera (index 2) ─┘                                  │
                                                           └─> pos_change.py
                                                               (next grid cell)

Serial controller (9600 baud) ──> f / c / n commands ──> main.py
```

All YOLO inference calls currently use a confidence threshold of `0.7`.

## Repository contents

| File | Purpose |
| --- | --- |
| `main.py` | Entry point. Opens the cameras and serial port, loads both YOLO models, processes zone 1 or zone 2, draws detections, and maintains navigation state. |
| `Cam_data.py` | Converts zone-2 detections into `left`, `mid`, and `right` classes by checking bounding-box centres in the middle horizontal band of each image. |
| `Row_data.py` | Maps the three visible classes into the 12-cell arena dictionary and decides whether to continue forward or change position. |
| `pos_change.py` | Updates current/next row, position, and column after the routing decision. |
| `spf3.pt` | Custom zone-1 YOLO checkpoint. Its stored class metadata contains `Fist`, `Palms`, `Plam`, and `Spearheads`. |
| `symbol300.pt` | Custom zone-2 YOLO checkpoint with the classes `R2Real` and `Rsymbol`. |
| `__pycache__/` | Generated Python bytecode; it is not required source code and can be recreated automatically. |

`Row_data.py` also recognises `Fake` as a blocked class, although `Fake` is not
present in the class metadata of the included `symbol300.pt` checkpoint.

## Arena and camera mapping

Zone 2 is represented by a 4-by-3 grid. Position numbers increase from the
physical right side to the physical left side:

```text
                  Physical view
              Left     Centre     Right
Row 1           3         2         1
Row 2           6         5         4
Row 3           9         8         7
Row 4          12        11        10
```

The left camera supplies the left and centre classifications. The right camera
supplies the right classification. Only detections whose centre satisfies
`160 < y < 320` are used, and the left/right split is at `x = 320`. These
coordinates and the display overlays assume approximately 640 x 480 frames.

Routing behaviour:

- `R2Real` or `empty`: continue to the cell in the next row and keep the same
  column.
- `Rsymbol` or `Fake`: select a neighbouring fallback cell and update the next
  column.
- Reaching positions 10-12 sets the next destination to `exit`.

## Requirements

### Hardware

- three cameras visible to OpenCV at indices `1`, `2`, and `3`;
- a serial-connected robot controller; and
- a display environment for the OpenCV preview windows.

### Software

- Python 3.10 or a compatible version;
- OpenCV;
- Ultralytics YOLO;
- NumPy; and
- pySerial.

Install the Python dependencies in your environment:

```bash
python -m pip install ultralytics opencv-python numpy pyserial
```

No dependency lock file is currently included, so record tested package
versions before deploying this program on the robot.

## Configuration

Edit the hardware settings near the top of `main.py` before running:

```python
left = cv2.VideoCapture(1)
right = cv2.VideoCapture(2)
sph = cv2.VideoCapture(3)

ser = serial.Serial(port="COMx", baudrate=9600, timeout=1)
```

Required configuration:

1. Replace `COMx` with the controller's real port, such as `COM5` on Windows or
   `/dev/ttyUSB0` on Linux.
2. Confirm the three OpenCV camera indices; their values depend on connection
   order and operating system.
3. Confirm that the frames are 640 x 480, or update the hard-coded grid
   coordinates and the zone-1 centre value (`320`).
4. Tune `FOV = 80` and the HSV red/green thresholds for the installed camera,
   lens, lighting, and arena colours.
5. Select the required processing state with `z1` and `z2`. The current defaults
   enable zone 1 only.

The model paths are relative, so run the entry point from this directory unless
the code is changed to resolve paths relative to `main.py`.

## Running

```bash
cd TKR_2026_IP_Codes
python main.py
```

When zone 2 is active, the program displays `left` and `right` preview windows.
Press `q` in a preview window to stop the loop.

## Serial command protocol

`main.py` reads one command with `ser.readline().decode()`:

| Command | Active zone | Behaviour |
| --- | --- | --- |
| `f` | Zone 1 | Confirms alignment when the detected target centre is between x = 260 and x = 380, and prints the estimated horizontal angle. |
| `c` | Zone 2 | Classifies the first row and chooses the first `R2Real` position. |
| `n` | Zone 2 | Advances the navigation state, analyses the next row, and calculates the next position/column. |

Send only the command character. If the controller terminates messages with
`\r` or `\n`, strip whitespace from the decoded signal before comparing it.

## Module behaviour

### `Cam_data.py`

- `Left_camera_Detection(...)` returns `(left_class, mid_class)`.
- `Right_camera_Detection(...)` returns `right_class`.
- A slot remains `empty` when no qualifying detection is found.
- If multiple boxes fall into one slot, the last iterated detection wins.

### `Row_data.py`

- `first_row(...)` stores the first three classifications in positions 1-3 and
  selects the first `R2Real` position in numeric order.
- `further_rows(...)` fills the next visible row, evaluates the cell ahead, and
  sets `pos_to_change` for the movement update.

### `pos_change.py`

- `pos_is_true(...)` advances the current position to the next row while
  retaining its column.
- `pos_is_false(...)` keeps the selected fallback position and changes the
  column used by the next detection cycle.


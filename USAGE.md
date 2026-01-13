# USAGE - Actual Code Usage Examples

This document shows how the functions and classes mentioned in [NEWS.md](NEWS.md) are actually used in the donkeycar codebase. All examples are extracted from the actual implementation, not generic examples.

---

## Table of Contents

1. [Course Analysis Framework](#1-course-analysis-framework)
2. [Segment-Based Training](#2-segment-based-training)
3. [IMU Path Visualization](#3-imu-path-visualization)
4. [Field Aggregations](#4-field-aggregations)
5. [CLI Commands](#5-cli-commands)
6. [BNO055 Sensor](#6-bno055-sensor)
7. [Pipeline Integration](#7-pipeline-integration)

---

## 1. Course Analysis Framework

### PathData Loading

**File:** `donkeycar/management/imupath.py`

```python
from donkeycar.course_analysis import (
    CSVPathDataSource,
    TubPathDataSource,
)

# Load from CSV file
if os.path.isfile(data_source) and data_source.endswith('.csv'):
    source = CSVPathDataSource(data_source)
elif os.path.isdir(data_source):
    source = TubPathDataSource(data_source)

path_data = source.load()
print(f"  {len(path_data.timestamp)} data points")
print(f"  Total distance: {path_data.total_distance:.2f}m")
print(f"  Duration: {path_data.duration:.2f}s")
```

### Lap Detection

**File:** `donkeycar/utilities/interactive_imu_viz.py`

```python
from donkeycar.course_analysis import (
    YCrossingLapDetector,
    DriftLapDetector,
    MultiLapData,
)

# Initialize detector based on method
if self.lap_method == 'y_crossing':
    detector = YCrossingLapDetector(params=lap_params)
elif self.lap_method == 'drift':
    detector = DriftLapDetector(params=lap_params)

# Detect laps from path data
class PathDataSource:
    def load(self):
        return path_data

source = PathDataSource()
self.multilap_data = MultiLapData.from_source(source, detector)
```

**File:** `donkeycar/parts/tub_statistics.py`

```python
from donkeycar.course_analysis import (
    TubPathDataSource,
    YCrossingLapDetector,
    DriftLapDetector,
)

# Create data source from tub
data_source = TubPathDataSource(self.tub.base_path)

# Select detector
if lap_detector == 'ycrossing':
    detector = YCrossingLapDetector()
elif lap_detector == 'drift':
    detector = DriftLapDetector()
else:
    raise ValueError(f'Unknown lap detector: {lap_detector}')

# Detect laps
multilap_data = MultiLapData.from_source(data_source, detector)
```

### Mean Course Building

**File:** `donkeycar/utilities/interactive_imu_viz.py`

```python
from donkeycar.course_analysis import MeanCourseBuilder

# Build mean course from selected number of laps
builder = MeanCourseBuilder(params=mean_params)
self.mean_course = builder.build(self.multilap_data)
```

**File:** `donkeycar/parts/tub_statistics.py`

```python
from donkeycar.course_analysis import MeanCourseBuilder

# Build mean course with default parameters
builder = MeanCourseBuilder()
mean_course = builder.build(multilap_data)
```

### Course Segmentation

**File:** `donkeycar/utilities/interactive_imu_viz.py`

```python
from donkeycar.course_analysis import (
    CourseSegmenter,
    ThresholdSegmentation,
    ExtremaSegmentation,
    GradientSegmentation,
    HybridSegmentation,
)

# Build segmentation parameters
seg_params = {
    'min_segment_length': min_segment_length,
    'curvature_threshold': curvature_threshold
}

# Create segmenter based on strategy
if self.segment_method == 'threshold':
    strategy = ThresholdSegmentation(params=seg_params)
elif self.segment_method == 'extrema':
    strategy = ExtremaSegmentation(params=seg_params)
elif self.segment_method == 'gradient':
    strategy = GradientSegmentation(params=seg_params)
elif self.segment_method == 'hybrid':
    strategy = HybridSegmentation(params=seg_params)

# Segment the mean course
segmenter = CourseSegmenter(strategy)
self.segmentation = segmenter.segment(self.mean_course)
```

**File:** `donkeycar/parts/tub_statistics.py`

```python
from donkeycar.course_analysis import CourseSegmentation

# Create segmentation from mean course
segmentation = CourseSegmentation.create(
    mean_course,
    strategy=segmentation_strategy,
    min_segment_length=min_segment_length,
    curvature_threshold=curvature_threshold
)
```

### Segment Assignment

**File:** `donkeycar/utilities/interactive_imu_viz.py`

```python
from donkeycar.course_analysis import SegmentAssigner

# Assign segments to full path
assigner = SegmentAssigner(self.segmentation)
self.segment_ids = assigner.assign(
    self.path_data.x,
    self.path_data.y
)
```

**File:** `donkeycar/parts/tub_statistics.py`

```python
from donkeycar.course_analysis import SegmentAssigner

# Assign segments to each record in tub
assigner = SegmentAssigner(segmentation)

for record in tub:
    # Get position from record
    pos = record.get('car/pos')
    if pos:
        x, y = pos[0], pos[1]
        segment_id = assigner.assign_single(x, y)
        record.update({'car/segment': segment_id})
```

### Integration Test Example

**File:** `donkeycar/tests/test_integration_course_analysis.py`

```python
from donkeycar.course_analysis import (
    PathData, CSVPathDataSource,
    YCrossingLapDetector, DriftLapDetector, MultiLapData,
    MeanCourseBuilder, MeanCourse,
    GradientSegmentation, CourseSegmenter, CourseSegmentation,
    SegmentAssigner
)

# Step 1: Load 3-lap data
path_data = create_synthetic_3lap_oval(points_per_lap=100)

# Step 2: Detect all laps
detector = YCrossingLapDetector()
multilap_data = MultiLapData.from_source(
    type('Source', (), {'load': lambda self: path_data})(),
    detector
)

# Step 3: Build mean from ONLY 2 laps (user selects this in UI)
limited_boundaries = multilap_data.lap_boundaries[:2]
limited_data = MultiLapData(path_data, limited_boundaries)

builder = MeanCourseBuilder()
mean_course = builder.build(limited_data)

# Step 4: Segment mean course
segmenter = CourseSegmenter(GradientSegmentation())
segmentation = segmenter.segment(mean_course)

# Step 5: Assign to FULL 3-lap path
assigner = SegmentAssigner(segmentation)
segment_ids = assigner.assign(path_data.x, path_data.y)

# Use ACTUAL detected boundary (not guessed!)
lap1_end = multilap_data.lap_boundaries[0].end_index
lap1_segments = set(segment_ids[:lap1_end + 1])

# Verify lap 1 has multiple segments (proves transitions work)
assert len(lap1_segments) > 1
```

---

## 2. Segment-Based Training

### TubStatistics Integration

**File:** `donkeycar/parts/tub_statistics.py`

```python
from donkeycar.parts.tub_v2 import Tub
from donkeycar.parts.tub_statistics import TubStatistics

# Initialize with tub and config
tub = Tub(tub_path, read_only=False)
stats = TubStatistics(tub, config=config)

# Compute segment assignments
stats.compute_segment_assignments(
    lap_detector='ycrossing',
    segmentation_strategy='hybrid',
    min_segment_length=1.0,
    curvature_threshold=0.1
)

# This writes 'car/segment' field to each tub record
# and stores segmentation metadata in session
```

### PctMode in Training Pipeline

**File:** `donkeycar/pipeline/types.py`

```python
from enum import Enum

class PctMode(Enum):
    """
    Performance ranking mode for training.

    NONE: No performance ranking (no lap_pct field)
    LAP: Lap-based performance ranking
    SEGMENT: Segment-based performance ranking
    """
    NONE = 0
    LAP = 1
    SEGMENT = 2
```

**File:** `donkeycar/pipeline/training.py`

```python
from donkeycar.pipeline.types import TubDataset, PctMode

# Auto-detect pct_mode from config
def determine_pct_mode(config):
    if getattr(config, 'SEGMENT_PCT_MODE', False):
        return PctMode.SEGMENT
    elif getattr(config, 'TRAIN_FILTER_PERCENT', 0) > 0:
        return PctMode.LAP
    else:
        return PctMode.NONE

pct_mode = determine_pct_mode(config)

# Create dataset with segment-based ranking
dataset = TubDataset(
    config=config,
    tub_paths=tub_paths,
    pct_mode=pct_mode
)
```

### Field Aggregations in TubStatistics

**File:** `donkeycar/parts/tub_statistics.py`

```python
from dataclasses import dataclass
from typing import Optional, Callable, Any

@dataclass
class FieldAggregationSpec:
    """
    Specification for a field aggregation.
    
    field: The field name in tub records (e.g., 'car/gyro')
    output_key: The key for the aggregated value (e.g., 'gyro_z_agg')
    index: Optional index for array fields (e.g., 2 for z-axis)
    transform: Optional transformation function (e.g., abs)
    aggregation: Aggregation method ('avg', 'sum', 'min', 'max', 'median', 'std')
    """
    field: str
    output_key: str
    index: Optional[int] = None
    transform: Optional[Callable[[Any], Any]] = None
    aggregation: str = 'avg'

# Load from config
def _load_field_aggregations_from_config(self, config):
    config_specs = getattr(config, 'FIELD_AGGREGATIONS', None)
    
    if not config_specs:
        # Fallback to legacy GYRO_Z_INDEX
        gyro_z_index = getattr(config, 'GYRO_Z_INDEX', 1)
        return [
            FieldAggregationSpec(
                field='car/gyro',
                output_key='gyro_z_agg',
                index=gyro_z_index,
                transform=abs,
                aggregation='avg'
            )
        ]
    
    # Convert config dicts to FieldAggregationSpec
    specs = []
    for spec_dict in config_specs:
        spec = FieldAggregationSpec(
            field=spec_dict['field'],
            output_key=spec_dict['output_key'],
            index=spec_dict.get('index'),
            transform=spec_dict.get('transform'),
            aggregation=spec_dict.get('aggregation', 'avg')
        )
        specs.append(spec)
    
    return specs
```

---

## 3. IMU Path Visualization

### InteractiveIMUVisualizer Usage

**File:** `donkeycar/management/imupath.py`

```python
from donkeycar.course_analysis import (
    CSVPathDataSource,
    TubPathDataSource,
)
from donkeycar.utilities.interactive_imu_viz import InteractiveIMUVisualizer

# Load data
if os.path.isfile(data_source) and data_source.endswith('.csv'):
    source = CSVPathDataSource(data_source)
elif os.path.isdir(data_source):
    source = TubPathDataSource(data_source)

path_data = source.load()

# Create visualizer
viz = InteractiveIMUVisualizer(
    path_data=path_data,
    cfg=None,
    lap_method='y_crossing',  # or 'drift'
    segment_method='gradient',  # or 'threshold', 'extrema', 'hybrid'
    file_path=data_source
)

# Setup and show UI
viz.setup_ui()
viz.show()
```

**File:** `donkeycar/utilities/interactive_imu_viz.py`

```python
from donkeycar.course_analysis import (
    YCrossingLapDetector,
    DriftLapDetector,
    MultiLapData,
    MeanCourseBuilder,
    CourseSegmenter,
    SegmentAssigner,
    ThresholdSegmentation,
    ExtremaSegmentation,
    GradientSegmentation,
    HybridSegmentation,
)

class InteractiveIMUVisualizer:
    def __init__(self, path_data, cfg, lap_method='y_crossing',
                 segment_method='gradient', file_path=''):
        # Store immutable data
        self.path_data = path_data
        self.cfg = cfg
        self.file_path = file_path
        
        # Create DataFrame for easier time-based indexing
        self.df = pd.DataFrame({
            't': path_data.timestamp,
            'x': path_data.x,
            'y': path_data.y,
            'h': path_data.heading,
            'v': path_data.velocity
        })
        
        # Current settings (mutable UI state)
        self.lap_method = lap_method
        self.segment_method = segment_method
        self.num_laps_for_mean = None
        
        # Data processing results
        self.multilap_data = None
        self.mean_course = None
        self.segmentation = None
        self.segment_ids = None
```

---

## 4. Field Aggregations

### Configuration

**File:** `donkeycar/templates/cfg_complete.py`

```python
# FIELD_AGGREGATIONS - Define custom performance metrics
# These metrics are aggregated per lap or segment for ranking
FIELD_AGGREGATIONS = [
    {
        'field': 'car/gyro',           # Field name from tub records
        'index': 2,                    # Z-axis (for array fields)
        'output_key': 'gyro_z_agg',   # Key for aggregated value
        'transform': abs,              # Transform function (e.g., abs, lambda x: x**2)
        'aggregation': 'avg'           # Aggregation method: avg, sum, min, max, median, std
    }
]

# LAP_SORTING_CRITERIA - Multi-criteria ranking
# These keys must match the 'output_key' values in FIELD_AGGREGATIONS
LAP_SORTING_CRITERIA = [
    {'key': 'time'},                   # Primary: lap time
    {'key': 'distance'},               # Secondary: distance traveled
    {'key': 'gyro_z_agg'},            # Tertiary: smoothness (gyro)
]
```

### Advanced Example - Multi-Objective Optimization

**File:** `donkeycar/templates/cfg_complete.py`

```python
# Example: Fast + Smooth + Low Lateral Acceleration
FIELD_AGGREGATIONS = [
    {
        'field': 'car/gyro',
        'index': 2,
        'output_key': 'gyro_z_agg',
        'transform': abs,
        'aggregation': 'avg'
    },
    {
        'field': 'car/accel',
        'index': 0,                    # X-axis acceleration
        'output_key': 'accel_x_sum',
        'transform': abs,
        'aggregation': 'sum'
    }
]

# Ranking: Fast (time), then smooth (low gyro), then gentle (low accel)
LAP_SORTING_CRITERIA = [
    {'key': 'time'},
    {'key': 'gyro_z_agg'},
    {'key': 'accel_x_sum'},
]
```

### Loading in TubStatistics

**File:** `donkeycar/parts/tub_statistics.py`

```python
# Initialize TubStatistics with config
stats = TubStatistics(tub, config=config)

# Field aggregations are loaded from config
# Priority order:
# 1. Explicit field_aggregations parameter (for testing)
# 2. From config (FIELD_AGGREGATIONS)
# 3. Default fallback (gyro_z with GYRO_Z_INDEX)

# During segment/lap iteration, accumulate field values
for record in tub:
    for spec in self.field_aggregations:
        value = record.get(spec.field)
        if value is not None:
            # Extract indexed value if needed
            if spec.index is not None and hasattr(value, '__getitem__'):
                value = value[spec.index]
            
            # Apply transform if specified
            if spec.transform:
                value = spec.transform(value)
            
            # Accumulate for aggregation
            field_accumulator.add(value)

# At segment/lap end, aggregate accumulated values
aggregated_value = field_accumulator.aggregate(spec.aggregation)
```

---

## 5. CLI Commands

### donkey imupath

**File:** `donkeycar/management/imupath.py`

```python
class ImuPathCommand:
    def parse_args(self, args):
        parser = argparse.ArgumentParser(
            prog='imupath',
            usage='%(prog)s [options] [data_source]',
            description='Visualize IMU path data')
        
        parser.add_argument('data_source', nargs='?', default='imu.csv',
                           help='path to CSV file or Tub directory')
        parser.add_argument('--lap-method', type=str,
                           choices=['y_crossing', 'drift'],
                           default='y_crossing',
                           help='lap detection method (default: y_crossing)')
        parser.add_argument('--segment-method', type=str,
                           choices=['threshold', 'extrema', 'gradient',
                                    'hybrid'],
                           default='gradient',
                           help='segmentation method (default: gradient)')
        parser.add_argument('--min-loop-distance', type=float, default=1.0,
                           help='minimum loop distance in meters')
        parser.add_argument('--num-laps', type=int, default=None,
                           help='number of laps for mean course (default: all)')
        
        return parser.parse_args(args)

# Actual usage examples:
# donkey imupath ./recordings/track_session.csv
# donkey imupath --lap-method drift --segment-method hybrid ./data/tub_1
# donkey imupath ./data/tub_1 --num-laps 3
```

### donkey segment

**File:** `donkeycar/management/segment.py`

```python
class SegmentCommand:
    def parse_args(self, args):
        parser = argparse.ArgumentParser(
            prog='segment',
            usage='%(prog)s [options] <tub_path>',
            description='Compute and assign segments to tub records')
        
        parser.add_argument('tub', type=str,
                           help='path to Tub directory')
        parser.add_argument('--lap-detector', type=str,
                           choices=['ycrossing', 'drift'],
                           default='ycrossing',
                           help='lap detection method (default: ycrossing)')
        parser.add_argument('--strategy', type=str,
                           choices=['threshold', 'extrema', 'gradient',
                                    'hybrid'],
                           default='hybrid',
                           help='segmentation strategy (default: hybrid)')
        parser.add_argument('--min-segment-length', type=float, default=1.0,
                           help='minimum segment length in meters (default: 1.0)')
        parser.add_argument('--curvature-threshold', type=float, default=0.1,
                           help='curvature threshold for segmentation (default: 0.1)')
        parser.add_argument('--config', type=str, default=None,
                           help='path to config file for field aggregations')
        
        return parser.parse_args(args)
    
    def run(self, args):
        args = self.parse_args(args)
        tub_path = os.path.expanduser(args.tub)
        
        # Load tub
        tub = Tub(tub_path, read_only=False)
        
        # Load config if provided
        config = None
        if args.config:
            config = load_config(args.config)
        
        # Compute segment assignments
        stats = TubStatistics(tub, config=config)
        stats.compute_segment_assignments(
            lap_detector=args.lap_detector,
            segmentation_strategy=args.strategy,
            min_segment_length=args.min_segment_length,
            curvature_threshold=args.curvature_threshold
        )

# Actual usage examples:
# donkey segment ./data/tub_1
# donkey segment --lap-detector ycrossing --strategy hybrid ./data/tub_1
# donkey segment --strategy hybrid --min-segment-length 1.5 ./data/tub_1
```

---

## 6. BNO055 Sensor

### BNO055Ada Implementation

**File:** `donkeycar/parts/imu.py`

```python
import board
import adafruit_bno055
import numpy as np
import math
import time
import logging

logger = logging.getLogger(__name__)

class BNO055Ada:
    def __init__(self, alpha=1.0, record_path=False, correction=None):
        # Initialize I2C and sensor
        i2c = board.I2C()  # uses board.SCL and board.SDA
        self.sensor = adafruit_bno055.BNO055_I2C(i2c)
        self.last_val = 0xFFFF
        
        # Set calibration offsets
        self.sensor.offsets_accelerometer = (41, -116, -26)
        self.sensor.offsets_gyroscope = (-1, -1, -1)
        self.sensor.offsets_magnetometer = (-176, 196, 17)
        
        # Initialize state
        self.pos = np.zeros(3)  # [x, y, z] where x=forward, y=left, z=up
        self.accel = np.zeros(3)
        self.gyro = np.array(self.sensor.gyro)
        self.path = []
        self.time = None
        self.on = True
        
        # Euler angles are in z, y, x order in the sensor
        self.euler = np.array(self.sensor.euler[::-1])
        self.heading = math.radians(90 - self.euler[2])
        self.alpha = alpha
        self.record_path = record_path
        self.correction = correction  # (corr_x, corr_y)
        self.odometer_speed = 0.0
        
        logger.info(f"Created BNO055, with alpha={self.alpha}, record_path="
                    f"{self.record_path} and correction={self.correction}")
    
    def poll(self):
        new_time = time.time()
        if self.time is None:
            self.time = new_time
        dt = new_time - self.time
        
        # Read gyro and ignore [0, 0, 0] sensor errors
        gyro_reading = np.array(self.sensor.gyro)
        if np.any(gyro_reading):
            self.gyro *= (1.0 - self.alpha)
            self.gyro += self.alpha * gyro_reading
        
        # Read accel and ignore [0, 0, 0] sensor errors
        accel_reading = np.array(self.sensor.linear_acceleration)
        if np.any(accel_reading):
            self.accel *= (1.0 - self.alpha)
            self.accel += self.alpha * accel_reading
        
        # Update state...
```

### Usage in Donkey5 Template

**File:** `donkeycar/templates/donkey5.py`

```python
from donkeycar.parts.imu import BNO055Ada

# Add IMU sensor to vehicle
imu = BNO055Ada(
    alpha=cfg.IMU_ALPHA,
    record_path=record_path,
    correction=cfg.IMU_CORRECTION
)

car.add(
    imu,
    inputs=['car/speed'],
    outputs=['car/pos', 'car/euler', 'car/gyro', 'car/accel'],
    threaded=True
)

# The sensor provides:
# - car/pos: [x, y, z] position in meters
# - car/euler: [roll, pitch, yaw] in degrees
# - car/gyro: [gx, gy, gz] angular velocity in rad/s
# - car/accel: [ax, ay, az] linear acceleration in m/s²
```

---

## 7. Pipeline Integration

### TubDataset with Segment Mode

**File:** `donkeycar/pipeline/types.py`

```python
from donkeycar.pipeline.types import TubDataset, PctMode

# Create dataset with segment-based performance ranking
dataset = TubDataset(
    config=config,
    tub_paths=['/path/to/tub_1', '/path/to/tub_2'],
    pct_mode=PctMode.SEGMENT  # or PctMode.LAP or PctMode.NONE
)

# Dataset will:
# 1. Load segment assignments from tub metadata
# 2. Calculate segment performance rankings
# 3. Filter records based on segment performance
# 4. Populate 'lap_pct' field in TubRecord from segment rankings
```

### BatchSequence in Training

**File:** `donkeycar/pipeline/training.py`

```python
from donkeycar.pipeline.types import TubDataset, TubRecord, PctMode

class BatchSequence(object):
    def __init__(self, model, config, records, is_train):
        self.model = model
        self.config = config
        self.records = records  # List[TubRecord]
        self.batch_size = self.config.BATCH_SIZE
        self.is_train = is_train
        # ... initialize augmentation and transformation
    
    def _create_pipeline(self):
        # 1. Define transformations
        def get_x(record: TubRecord):
            """Extract x from record for training"""
            out_dict = self.model.x_transform(record, self.image_processor)
            out_dict['img_in'] = normalize_image(out_dict['img_in'])
            return out_dict
        
        def get_y(record: TubRecord):
            """Extract y from record for training"""
            y = self.model.y_transform(record)
            return y
        
        def get_w(record: TubRecord):
            """Extract sample weights if using weighted training"""
            w = self.model.w_transform(record)
            return w
        
        # 2. Build pipeline using the transformations
        use_weights = getattr(self.config, 'LAP_QUANTIFIER', '').lower() \
            == 'weight'
        w_transform = get_w if use_weights else None
        
        pipeline = PipelineGenerator(
            self.records,
            x_transform=get_x,
            y_transform=get_y,
            w_transform=w_transform
        )
        return pipeline

# The TubRecord objects have been populated with 'lap_pct' field
# based on segment performance when SEGMENT_PCT_MODE = True
```

### Configuration for Segment Training

**File:** `donkeycar/templates/cfg_complete.py`

```python
# Enable segment-based performance training
SEGMENT_PCT_MODE = False  # Set to True to enable

# Segmentation parameters
SEGMENT_STRATEGY = 'hybrid'  # threshold, extrema, gradient, or hybrid
SEGMENT_LAP_DETECTOR = 'ycrossing'  # ycrossing or drift
SEGMENT_MIN_LENGTH = 1.0  # Minimum segment length in meters
SEGMENT_CURVATURE_THRESHOLD = 0.1  # Curvature threshold for segmentation

# Field aggregations for performance metrics
FIELD_AGGREGATIONS = [
    {
        'field': 'car/gyro',
        'index': 2,                    # Z-axis
        'output_key': 'gyro_z_agg',
        'transform': abs,
        'aggregation': 'avg'
    }
]

# Sorting criteria (primary to tertiary)
LAP_SORTING_CRITERIA = [
    {'key': 'time'},                   # Primary: lap time
    {'key': 'distance'},               # Secondary: distance
    {'key': 'gyro_z_agg'},            # Tertiary: smoothness
]
```

### Complete Training Workflow

**File:** Combined from multiple files

```bash
# 1. Create car with IMU support
donkey createcar --template donkey5 --path ~/mycar
cd ~/mycar

# 2. Record training data (3-5 laps with IMU enabled)
python manage.py drive --tub ./data

# 3. Compute segment assignments
donkey segment --tub ./data/tub_1 --strategy hybrid

# 4. Configure training (edit myconfig.py)
# SEGMENT_PCT_MODE = True
# SEGMENT_STRATEGY = 'hybrid'

# 5. Train model with segment-based performance
python manage.py train --tub ./data/tub_1 --model ./models/pilot.h5
```

---

## Summary

All examples in this document are extracted from actual implementation files in the donkeycar codebase. The key locations are:

- **Course Analysis**: `donkeycar/course_analysis/`, `donkeycar/utilities/interactive_imu_viz.py`
- **Segment Training**: `donkeycar/parts/tub_statistics.py`, `donkeycar/pipeline/training.py`
- **CLI Commands**: `donkeycar/management/imupath.py`, `donkeycar/management/segment.py`
- **IMU Sensor**: `donkeycar/parts/imu.py`, `donkeycar/templates/donkey5.py`
- **Configuration**: `donkeycar/templates/cfg_complete.py`
- **Tests**: `donkeycar/tests/test_integration_course_analysis.py`

For more details, see the [NEWS.md](NEWS.md) file and the individual chapter documentation files.

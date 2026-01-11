# NEWS - DocGarbanzo Fork Enhancements

This document outlines the major differences and enhancements in the 
DocGarbanzo fork compared to the original autorope/donkeycar repository.

## Overview

This fork extends the Donkey Car platform with advanced course analysis, 
segment-based training, and enhanced IMU sensor capabilities. The focus is on 
improving autonomous driving performance through intelligent data analysis and 
training methodologies.

---

## Major Features

### 1. Course Analysis Framework

A comprehensive course analysis system for analyzing recorded driving data and 
computing geometric track representations.

**Location:** `donkeycar/course_analysis/`

**Key Components:**
- **Data Loading** (`data_loader.py`)
  - `PathData` container for trajectory data (position, heading, velocity)
  - `CSVPathDataSource` - Load from CSV files (t, x, y, h, v format)
  - `TubPathDataSource` - Load from tub directories
  
- **Lap Detection** (`lap_detection.py`)
  - `YCrossingLapDetector` - Detects laps based on Y-coordinate crossings
  - `DriftLapDetector` - Handles GPS drift using cluster analysis
  
- **Mean Course Computation** (`mean_course.py`)
  - `MeanCourseBuilder` - Computes reference course from multiple laps
  - Handles alignment and averaging of lap trajectories
  
- **Course Segmentation** (`segmentation.py`)
  - Four segmentation strategies:
    - `threshold` - Distance-based segments
    - `extrema` - Curvature extrema-based segments
    - `gradient` - Curvature gradient-based segments
    - `hybrid` - Combined approach (recommended)
  - Divides course into geometric features (straights, curves, transitions)
  
- **Segment Assignment** (`segment_assignment.py`)
  - `SegmentAssigner` - Assigns trajectory points to course segments
  - Two-stage approach:
    1. Nearest-neighbor for initial detection
    2. Tangent projection for crossing detection

**Benefits:**
- Understand track geometry and driving characteristics
- Identify consistent course features across laps
- Enable segment-based performance analysis

---

### 2. Segment-Based Performance Training

Train models on the best-driven instances of each track segment across all 
laps, rather than just the best complete laps. This creates a "synthetic 
perfect lap" that outperforms any single recorded lap.

**Key Files:**
- `donkeycar/pipeline/types.py` - Added `PctMode` enum and segment support
- `donkeycar/parts/tub_statistics.py` - Segment performance calculation
- `donkeycar/pipeline/training.py` - Training pipeline integration

**Configuration:**
```python
# In cfg_complete.py
SEGMENT_PCT_MODE = False  # True = segment-based, False = lap-based
SEGMENT_STRATEGY = 'hybrid'  # threshold, extrema, gradient, or hybrid
SEGMENT_LAP_DETECTOR = 'ycrossing'  # ycrossing or drift
SEGMENT_MIN_LENGTH = 1.0  # Minimum segment length in meters
SEGMENT_CURVATURE_THRESHOLD = 0.1  # Curvature threshold for segmentation
```

**Workflow:**
1. Record multi-lap driving data with IMU enabled
2. Compute segment assignments: `donkey segment --tub ./data/tub_1`
3. Enable segment mode in config: `SEGMENT_PCT_MODE = True`
4. Train model: `python manage.py train --tub ./data/tub_1`

**How It Works:**
- Each tub record gets a `car/segment` field (integer segment ID)
- Segment performance rankings stored as: 
  `session_rank[session_id][lap_num][segment_id] = [time_pct, gyro_z_pct, 
  distance_pct]`
- Training prioritizes the fastest/smoothest instance of each segment across 
  all laps
- Model learns from "best-of-breed" segments rather than complete laps

**Example:**
```
3 laps, 4 segments per lap:

Lap 1: Segments [Fast, Slow, Medium, Fast]
Lap 2: Segments [Medium, Fast, Fast, Slow]
Lap 3: Segments [Slow, Medium, Slow, Medium]

Training uses:
- Segment 0 from Lap 1 (fastest instance)
- Segment 1 from Lap 2 (fastest instance)
- Segment 2 from Lap 2 (fastest instance)
- Segment 3 from Lap 1 (fastest instance)

Result: A "synthetic perfect lap" better than any single lap!
```

**Iterative Improvement:**
1. Train with segment_pct on initial multi-lap data
2. Drive with trained model (performs better in some segments)
3. Collect new data from model-driven laps
4. Re-segment combined data (original + new laps)
5. Retrain - model learns from new best segments
6. Repeat - iteratively improve beyond initial human best lap

**Benefits:**
- Learn from best driving in each part of the track
- Overcome inconsistent lap performance
- Iteratively improve beyond human baseline
- More efficient use of training data

---

### 3. IMU Path Visualization

Interactive visualization tool for analyzing recorded vehicle trajectories with 
real-time lap detection and course segmentation.

**Command:** `donkey imupath <data_source>`

**Location:** `donkeycar/utilities/interactive_imu_viz.py`

**Features:**
- Interactive matplotlib interface with time slider
- Lap selection and highlighting
- Real-time segmentation with method selection
- Mean course computation and display
- Supports both CSV and Tub data sources

**Usage Examples:**
```bash
# Visualize CSV data
donkey imupath ./recording.csv

# Visualize tub data
donkey imupath ./data/tub_1

# Specify methods
donkey imupath --lap-method drift --segment-method hybrid ./data.csv

# Use specific number of laps for mean course
donkey imupath --num-laps 2 ./data/tub_1
```

**Data Format:**
- CSV: `t, x, y, h, v` (timestamp, position x/y in meters, heading in degrees, 
  velocity in m/s)
- Tub: Extracts from `_timestamp_ms`, `car/pos`, `car/euler`, `car/speed`

**Benefits:**
- Visual debugging of trajectory data
- Understand lap detection and segmentation behavior
- Verify course analysis parameters
- Compare different segmentation strategies

---

### 4. Segment Assignment Command

Compute and store segment assignments directly in tub data for later use in 
training.

**Command:** `donkey segment <tub_path>`

**Location:** `donkeycar/management/segment.py`

**Features:**
- Processes entire tub with lap detection and segmentation
- Writes `car/segment` field to each tub record
- Stores segmentation metadata in tub manifest
- Configurable parameters for lap detection and segmentation

**Usage Examples:**
```bash
# Basic segmentation
donkey segment ./data/tub_1

# Specify strategies
donkey segment --lap-detector ycrossing --strategy hybrid ./data/tub_1

# Custom parameters
donkey segment --min-segment-length 1.0 --curvature-threshold 0.1 ./data/tub_1

# With visualization
donkey segment --visualize ./data/tub_1
```

**Output:**
- Adds `car/segment` field to each record (integer segment ID)
- Stores metadata:
  ```python
  {
      'num_segments': int,
      'mean_course_params': {...},
      'segmentation_params': {...}
  }
  ```

**Benefits:**
- Pre-compute segments once, use for multiple training runs
- Consistent segmentation across training sessions
- Metadata tracking for reproducibility

---

### 5. BNO055 IMU Sensor Improvements

Enhanced error handling for the BNO055 9-axis IMU sensor to prevent data 
corruption from transient sensor errors.

**Location:** `donkeycar/parts/imu.py`

**Changes:**
- Added `np.any()` guard to gyro readings (previously only on euler angles)
- Added `np.any()` guard to accelerometer readings
- Prevents [0, 0, 0] error values from corrupting tub data

**Problem Solved:**
The BNO055 sensor occasionally returns [0, 0, 0] as a transient error 
condition. Previously, this only affected euler angle readings, but gyro and 
accelerometer data would be corrupted with zeros during actual turning or 
acceleration.

**Fix:**
```python
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

# Read euler angles and ignore [0, 0, 0] sensor errors
euler_reading = np.array(self.sensor.euler)
if np.any(euler_reading):
    self.euler *= (1.0 - self.alpha)
    self.euler += self.alpha * euler_reading
```

**Impact:**
- Prevents sporadic zero values in `car/gyro` and `car/accel` tub data
- Maintains last valid reading during sensor errors
- Improves data quality for IMU-based training

---

### 6. Donkey5 Template

New vehicle template optimized for RC controller operation and advanced sensor 
integration.

**Location:** `donkeycar/templates/donkey5.py`, 
`donkeycar/templates/cfg_donkey5.py`

**Features:**
- RC controller-first design (with optional web control)
- IMU path recording (`--record_path` flag)
- Enhanced logging configuration
- Modular part architecture
- Support for custom logging.conf

**Usage:**
```bash
# Create car with donkey5 template
donkey createcar --template donkey5 --path ~/mycar

# Run with path recording
cd ~/mycar
./manage.py drive --record_path

# Calibrate RC
./manage.py calibrate
```

**Benefits:**
- Optimized for physical RC car deployment
- Better integration with IMU sensors
- Flexible logging for debugging
- Cleaner separation of concerns

---

### 7. Field Aggregations for Performance Metrics

Configurable field aggregations allow custom behavioral parameters for lap and 
segment performance ranking.

**Location:** `donkeycar/templates/cfg_complete.py`

**Configuration:**
```python
# Define transform functions
def abs_transform(value):
    """Absolute value transform."""
    return abs(value)

# Configure field aggregations
FIELD_AGGREGATIONS = [
    {
        'field': 'car/gyro',
        'index': 2,                    # Z-axis
        'output_key': 'gyro_z_agg',
        'transform': abs_transform,
        'aggregation': 'avg'           # Options: avg, sum, min, max, median
    }
]

# Sorting criteria for ranking
LAP_SORTING_CRITERIA = [
    {'key': 'time'},                   # Primary: lap time
    {'key': 'distance'},               # Secondary: distance traveled
    {'key': 'gyro_z_agg'},            # Tertiary: smoothness (avg abs gyro_z)
]
```

**Features:**
- Extract and transform arbitrary tub fields
- Aggregate per lap or segment (avg, sum, min, max, median)
- Multi-criteria ranking (time, distance, smoothness, etc.)
- Extensible for custom metrics

**Benefits:**
- Train on multiple behavioral objectives (fast + smooth driving)
- Customize ranking to match competition rules
- Experiment with different performance metrics
- Support domain-specific requirements

---

## Testing Infrastructure

### Comprehensive Test Coverage

**Course Analysis Tests:**
- `test_data_loader.py` - PathData loading from CSV and Tub
- `test_integration_course_analysis.py` - End-to-end course analysis
- `test_segment_assignment.py` - Segment assignment algorithms
- `test_segment_estimator.py` - Segment estimation logic
- `test_segment_identification_multilap.py` - Multi-lap segmentation
- `test_segment_initial_detection.py` - Initial segment detection

**Segment Training Tests:**
- `test_segment_performance.py` - Segment performance calculation
- `test_course_segmentation_integration.py` - Segmentation integration
- `test_segment_training_integration.py` - End-to-end training
- `test_lap_pct_regression.py` - Backward compatibility

**Test Fixtures:**
- `course_test_fixtures.py` - Reusable test data and utilities

**Benefits:**
- Validates correctness of complex algorithms
- Prevents regressions
- Documents expected behavior
- Enables confident refactoring

---

## Documentation Enhancements

### CLAUDE.md

Comprehensive development guide for AI assistants working on the codebase, 
including:

**Testing Guidelines:**
- Integration testing requirements
- Test anti-patterns to avoid
- Required test workflow for bug fixes
- Examples of good vs. bad tests

**Architecture Documentation:**
- IMU Path Visualization system
- Segment-Based Performance training
- Course analysis workflow
- Critical design constraints

**Development Patterns:**
- Parts-based system (threaded vs non-threaded)
- Configuration system
- Remote development workflow (Raspberry Pi)
- Logging configuration

**Benefits:**
- Faster onboarding for new developers
- Consistent development patterns
- Reduced bugs through better testing
- Clear architectural decisions

---

### SEGMENT_IMPLEMENTATION_PLAN.md

Detailed implementation plan for segment-based performance feature, including:
- Phase-by-phase checklist
- File-by-file changes
- Data structure specifications
- Validation checklist
- Expected behavior examples

**Benefits:**
- Clear roadmap for complex feature
- Track implementation progress
- Reference for future enhancements
- Documentation of design decisions

---

## Configuration Changes

### Enhanced Configuration (cfg_complete.py)

**New Sections:**
1. **Segment Performance** (lines 765-772)
   - `SEGMENT_PCT_MODE` - Enable segment-based training
   - `SEGMENT_STRATEGY` - Segmentation method
   - `SEGMENT_LAP_DETECTOR` - Lap detection method
   - `SEGMENT_MIN_LENGTH` - Minimum segment length
   - `SEGMENT_CURVATURE_THRESHOLD` - Curvature threshold

2. **Field Aggregations** (lines 774-803)
   - `FIELD_AGGREGATIONS` - Custom metric definitions
   - `LAP_SORTING_CRITERIA` - Multi-criteria ranking
   - Extensible framework for behavioral parameters

**Benefits:**
- Centralized configuration for new features
- Clear documentation of options
- Backward compatible defaults

---

## Management Commands

### New Commands

1. **`donkey imupath`** - Interactive IMU visualization
   - Visualize recorded trajectories
   - Real-time lap detection and segmentation
   - Support CSV and Tub data sources

2. **`donkey segment`** - Compute segment assignments
   - Process tub data with segmentation
   - Store segments in tub records
   - Configurable strategies and parameters

**Benefits:**
- User-friendly CLI interface
- Consistent with existing donkey commands
- Well-documented with help text

---

## Pipeline Enhancements

### PctMode Enum

**Location:** `donkeycar/pipeline/types.py`

```python
class PctMode(Enum):
    NONE = 0     # No performance ranking
    LAP = 1      # Lap-based ranking (original)
    SEGMENT = 2  # Segment-based ranking (new)
```

**Integration:**
- `TubDataset` constructor accepts `pct_mode` parameter
- Training pipeline auto-detects mode from config
- Backward compatible with existing code

**Benefits:**
- Type-safe mode selection
- Clear separation of ranking strategies
- Easy to extend for future modes

---

## Summary of Benefits

### For Developers
- **Better Tools:** IMU visualization, segment analysis
- **Cleaner Code:** Modular course analysis framework
- **Better Testing:** Comprehensive test coverage
- **Better Docs:** CLAUDE.md, implementation plans

### For Users
- **Better Performance:** Segment-based training outperforms lap-based
- **Better Data Quality:** BNO055 error handling
- **More Control:** Configurable field aggregations
- **Easier Debugging:** Interactive visualization tools

### For Researchers
- **Extensible Framework:** Add new segmentation strategies
- **Custom Metrics:** Define domain-specific performance measures
- **Reproducible Results:** Metadata tracking in tubs
- **Iterative Improvement:** Segment-based training enables progressive 
  enhancement

---

## Migration Guide

### From Original Donkeycar

1. **Existing Features Still Work**
   - All original donkeycar features are preserved
   - Backward compatible with existing tubs and models
   - Default configuration matches original behavior

2. **Enabling New Features**
   ```python
   # In myconfig.py

   # Enable segment-based training
   SEGMENT_PCT_MODE = True
   SEGMENT_STRATEGY = 'hybrid'

   # Add custom metrics
   FIELD_AGGREGATIONS = [
       {
           'field': 'car/gyro',
           'index': 2,
           'output_key': 'gyro_z_agg',
           'transform': abs,
           'aggregation': 'avg'
       }
   ]
   ```

3. **Using New Commands**
   ```bash
   # Visualize your data
   donkey imupath ./data/tub_1

   # Compute segments
   donkey segment ./data/tub_1

   # Train with segments
   python manage.py train --tub ./data/tub_1
   ```

---

## Future Enhancements

Potential areas for further development:
- Real-time segmentation during driving
- Online learning from segment performance
- Multi-session segment analysis
- Advanced segmentation strategies
- Integration with path following
- Segment-based model interpretability

---

## Credits

**Primary Author:** DocGarbanzo (dirk.prange@web.de)

**Contributors:**
- Claude Sonnet 4.5 (AI pair programming)

**Based on:** autorope/donkeycar

---

## Version History

### Current Version (Fork)
- Course analysis framework
- Segment-based performance training
- IMU path visualization
- BNO055 sensor improvements
- Donkey5 template
- Field aggregations
- Comprehensive testing
- Enhanced documentation

### Base Version
- Based on autorope/donkeycar main branch
- Preserves all original features and compatibility

---

For detailed usage instructions and examples, see:
- `CLAUDE.md` - Development guide
- `SEGMENT_IMPLEMENTATION_PLAN.md` - Implementation details
- `donkeycar/templates/cfg_complete.py` - Configuration reference
- `donkeycar/course_analysis/` - API documentation

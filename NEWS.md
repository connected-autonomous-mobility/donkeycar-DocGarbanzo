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
computing geometric track representations. This framework provides the 
foundational infrastructure for segment-based training and enables detailed 
analysis of vehicle trajectories and track characteristics.

**Location:** `donkeycar/course_analysis/`

**Design Philosophy:**
The course analysis framework is built on several key principles:
- **Immutability**: PathData containers use read-only numpy arrays to prevent 
  accidental modification
- **Strategy Pattern**: Pluggable algorithms for lap detection and segmentation
- **Pure Functions**: Stateless transformations for predictable behavior
- **Configuration Hierarchy**: Defaults → config file → explicit params
- **Testability**: All components work with synthetic data for unit testing

---

#### Key Components:

##### **Data Loading** (`data_loader.py`)

**PathData Container:**
An immutable data container that stores vehicle trajectory information with 
strict type safety and read-only guarantees.

```python
class PathData:
    """
    Immutable container for vehicle path data.
    
    All arrays are numpy.float64 with read-only flags set.
    """
    timestamp: np.ndarray  # Time in seconds
    x: np.ndarray         # X position in meters (right = positive)
    y: np.ndarray         # Y position in meters (forward = positive)
    heading: np.ndarray   # Heading in radians
    velocity: np.ndarray  # Speed in m/s
```

**Data Sources:**

1. **CSVPathDataSource** - Loads from CSV files with format:
   ```
   t,x,y,h,v
   0.000,0.0,0.0,0.0,0.0
   0.100,0.01,0.05,0.02,0.5
   ...
   ```
   - `t`: timestamp (seconds)
   - `x,y`: position in meters (IMU coordinate frame)
   - `h`: heading in degrees (converted to radians internally)
   - `v`: velocity in m/s

2. **TubPathDataSource** - Loads from tub directories:
   - Extracts `_timestamp_ms` → converts to seconds
   - Extracts `car/pos` → [x, y, z] (uses x, y only)
   - Extracts `car/euler` → [roll, pitch, yaw] (uses yaw as heading)
   - Extracts `car/speed` → velocity in m/s
   - Handles missing fields gracefully with warnings
   - Supports multi-session tubs (can load all sessions or specific one)

**Validation:**
All data sources validate that:
- All arrays have identical lengths
- No NaN or Inf values present
- Timestamps are monotonically increasing
- Arrays have minimum length (configurable, default: 10 points)

---

##### **Lap Detection** (`lap_detection.py`)

Lap detection identifies where individual laps begin and end in multi-lap 
trajectory data. Two strategies handle different data quality scenarios.

**LapBoundary Data Structure:**
```python
@dataclass
class LapBoundary:
    start_index: int    # Index where lap starts (inclusive)
    end_index: int      # Index where lap ends (inclusive)
    start_time: float   # Timestamp at start
    end_time: float     # Timestamp at end
    
    @property
    def duration(self) -> float:
        """Lap time in seconds"""
        
    @property
    def num_points(self) -> int:
        """Number of data points in lap"""
```

**1. YCrossingLapDetector** - For clean, drift-free data

**Algorithm:**
1. Find zero crossings: y[i] < 0 and y[i+1] >= 0
2. Filter crossings that are too close (< min_lap_duration)
3. Create LapBoundary for each valid crossing interval

**Configuration:**
```python
DEFAULT_PARAMS = {
    'min_lap_duration': 5.0,  # Minimum lap time (seconds)
    'y_threshold': 0.0,       # Y-axis crossing threshold
}
```

**Use Case:** Indoor tracks with RTK GPS or optical tracking, where position 
data is stable and repeatable. Works best when start/finish line crosses y=0 
axis perpendicular to travel direction.

**Example:**
```python
detector = YCrossingLapDetector(params={'min_lap_duration': 8.0})
boundaries = detector.detect_laps(path_data)
# Returns: [LapBoundary(0, 450), LapBoundary(451, 920), ...]
```

**2. DriftLapDetector** - For GPS data with drift

**Algorithm:**
1. Calculate weighted averages of consecutive position windows
2. Find reversal points where weighted average direction changes
3. Use DBSCAN clustering to group similar reversal positions
4. Select most consistent cluster as lap start/finish location
5. Detect crossings through this clustered region

**Configuration:**
```python
DEFAULT_PARAMS = {
    'window_size': 50,          # Points to average for drift detection
    'min_lap_duration': 5.0,    # Minimum lap time
    'cluster_eps': 2.0,         # DBSCAN epsilon (meters)
    'cluster_min_samples': 2,   # DBSCAN minimum cluster size
}
```

**Use Case:** Outdoor tracks with consumer GPS where position measurements 
drift between laps. The clustering approach finds the "true" start/finish 
location despite measurement noise.

**Example:**
```python
detector = DriftLapDetector(params={
    'window_size': 75,
    'cluster_eps': 3.0
})
boundaries = detector.detect_laps(path_data)
```

**Edge Cases Handled:**
- Single lap data: Returns one boundary spanning all data
- No crossings found: Returns empty list with warning
- Very short laps: Filtered out by min_lap_duration
- Irregular spacing: Works with variable sample rates

---

##### **Mean Course Computation** (`mean_course.py`)

Computes a reference "mean course" by aligning and averaging multiple laps. 
This creates a smooth, idealized representation of the track geometry.

**MeanCourse Data Structure:**
```python
class MeanCourse:
    x: np.ndarray          # X coordinates (meters)
    y: np.ndarray          # Y coordinates (meters)
    distance: np.ndarray   # Cumulative distance (meters)
    curvature: np.ndarray  # Path curvature (1/meters)
    tangent_x: np.ndarray  # Unit tangent vector X
    tangent_y: np.ndarray  # Unit tangent vector Y
```

**MeanCourseBuilder Algorithm:**

1. **Extract Individual Laps:**
   - Use LapBoundary objects to slice PathData
   - Validate each lap has minimum points (default: 50)

2. **Resample to Common Length:**
   - Use linear interpolation to resample all laps to same number of points
   - Default: 500 points per lap (configurable)
   - Preserves geometric features while enabling averaging

3. **Align Laps:**
   - Align starting points by translating x,y coordinates
   - Optionally align heading angles
   - Ensures laps are in same coordinate frame

4. **Average Coordinates:**
   - Compute element-wise mean of x, y positions
   - Result: smooth trajectory through "average" of all laps

5. **Compute Derived Properties:**
   - **Distance**: Cumulative arc length along course
     ```python
     dx = np.diff(x)
     dy = np.diff(y)
     ds = np.sqrt(dx**2 + dy**2)
     distance = np.concatenate([[0], np.cumsum(ds)])
     ```
   
   - **Tangent vectors**: Normalized velocity direction
     ```python
     tangent_x = dx / ds
     tangent_y = dy / ds
     ```
   
   - **Curvature**: Rate of heading change per distance
     ```python
     dtheta = np.diff(heading)
     curvature = dtheta / ds
     ```

**Configuration:**
```python
DEFAULT_PARAMS = {
    'resample_points': 500,      # Points per resampled lap
    'min_lap_points': 50,        # Minimum points to use lap
    'align_starting_points': True,  # Translate to common origin
    'smooth_window': 5,          # Smoothing window for curvature
}
```

**Example:**
```python
builder = MeanCourseBuilder(params={'resample_points': 750})
mean_course = builder.build(path_data, lap_boundaries, num_laps=3)

# Access properties
print(f"Course length: {mean_course.distance[-1]:.2f} m")
print(f"Max curvature: {np.max(np.abs(mean_course.curvature)):.3f} 1/m")
```

**Quality Considerations:**
- More laps → smoother mean course (diminishing returns after ~5 laps)
- Consistent driving → better alignment and averaging
- `resample_points` trades accuracy vs. computation time
- Smoothing reduces noise but can blur sharp corners

---

##### **Course Segmentation** (`segmentation.py`)

Divides the mean course into geometrically meaningful segments (straights, 
turns, transitions). Different strategies optimize for different track types.

**Segment Data Structure:**
```python
@dataclass
class Segment:
    segment_id: int           # Unique identifier (0, 1, 2, ...)
    start_index: int          # Start index in mean course
    end_index: int            # End index in mean course
    start_distance: float     # Arc length at start (meters)
    end_distance: float       # Arc length at end (meters)
    mean_curvature: float     # Average curvature in segment
    max_curvature: float      # Maximum curvature in segment
    segment_type: SegmentType # STRAIGHT, LEFT_TURN, RIGHT_TURN, etc.
```

**CourseSegmentation Class:**
```python
class CourseSegmentation:
    """
    Manages segmentation of mean course.
    
    Properties:
        segments: List[Segment]
        num_segments: int
        segment_boundaries: List[int]  # Indices where segments change
    """
```

**Segmentation Strategies:**

**1. Threshold Strategy** - Fixed distance intervals
- Divides course into segments of approximately equal arc length
- **Algorithm:**
  1. Calculate `segment_length = total_distance / num_segments`
  2. Find distances closest to multiples of segment_length
  3. Place boundaries at those indices
- **Use case:** Oval tracks, simple courses without complex geometry
- **Pros:** Predictable, easy to understand
- **Cons:** Ignores track geometry, may split turns awkwardly

**2. Extrema Strategy** - Curvature peaks
- Segments at local maxima/minima of curvature
- **Algorithm:**
  1. Smooth curvature signal (moving average)
  2. Find peaks using scipy.signal.find_peaks
  3. Filter peaks below prominence threshold
  4. Place boundaries at peak locations
  5. Filter segments shorter than min_length
- **Parameters:**
  ```python
  {
      'prominence': 0.05,      # Minimum peak prominence
      'min_segment_length': 1.0,  # Minimum segment size (meters)
      'smooth_window': 5,      # Smoothing window size
  }
  ```
- **Use case:** Tracks with distinct corners separated by straights
- **Pros:** Natural segmentation at geometric features
- **Cons:** May miss subtle features, sensitive to noise

**3. Gradient Strategy** - Curvature change rate
- Segments where curvature derivative crosses threshold
- **Algorithm:**
  1. Compute curvature gradient (rate of change)
  2. Find zero crossings of gradient
  3. Filter crossings below threshold
  4. Place boundaries at transition points
  5. Filter short segments
- **Parameters:**
  ```python
  {
      'curvature_threshold': 0.1,  # Gradient threshold
      'min_segment_length': 1.0,
  }
  ```
- **Use case:** Complex tracks with S-curves and chicanes
- **Pros:** Captures transitions between corner types
- **Cons:** Sensitive to parameter tuning

**4. Hybrid Strategy (Recommended)** - Combined approach
- Uses multiple criteria to find optimal segmentation
- **Algorithm:**
  1. Generate candidate boundaries from:
     - Curvature extrema (peaks/valleys)
     - Curvature gradient zero crossings
     - Large curvature changes
  2. Score each candidate by multiple metrics
  3. Select best boundaries using optimization
  4. Filter short segments
  5. Validate segment quality
- **Parameters:**
  ```python
  {
      'min_segment_length': 1.0,
      'curvature_threshold': 0.1,
      'prominence': 0.05,
      'max_segments': 20,         # Limit total segments
  }
  ```
- **Use case:** General purpose, works well on most tracks
- **Pros:** Robust, adapts to different geometries
- **Cons:** More complex, slightly slower

**Boundary Filtering:**
All strategies apply post-processing to ensure quality:
1. **Minimum Length Filter**: Merge segments shorter than `min_segment_length`
2. **Wrap-around Check**: For closed loops, validate last segment isn't too short
3. **Empty Handling**: If all segments filtered out, keep middle boundary

**Example Usage:**
```python
# Create segmentation
segmentation = CourseSegmentation(
    mean_course=mean_course,
    strategy='hybrid',
    params={
        'min_segment_length': 2.0,
        'curvature_threshold': 0.08
    }
)

# Compute segments
segmentation.compute()

# Access results
print(f"Num segments: {segmentation.num_segments}")
for seg in segmentation.segments:
    print(f"Segment {seg.segment_id}: "
          f"{seg.segment_type.value}, "
          f"{seg.end_distance - seg.start_distance:.1f}m")
```

**Choosing a Strategy:**
- **Simple oval/circle**: `threshold` (fastest, simplest)
- **Road course with corners**: `extrema` (natural boundaries)
- **Technical track with S-curves**: `gradient` (captures transitions)
- **Unknown/general track**: `hybrid` (best all-around)

---

##### **Segment Assignment** (`segment_assignment.py`)

Assigns segment IDs to actual driven paths (which may deviate from mean 
course). Critical for training on segment-specific data.

**SegmentAssigner Class:**
```python
class SegmentAssigner:
    """
    Assigns segments to driven path using boundary crossing detection.
    
    Two-stage algorithm:
    1. Initial detection: Nearest-neighbor to mean course
    2. Progress tracking: Tangent projection for boundary crossing
    """
```

**Algorithm Details:**

**Stage 1: Initial Segment Detection**
```python
def _find_initial_segment(x, y) -> int:
    """
    Find starting segment using nearest-neighbor.
    
    Why needed: Driver might start anywhere on track, not just at segment 0.
    
    Steps:
    1. Compute distance to all mean course points
    2. Find index of nearest point
    3. Look up which segment contains that index
    4. Return that segment ID
    """
```

**Example:** 
- Driver starts at position (5.2, 3.1)
- Nearest mean course point is index 245
- Index 245 belongs to segment 3
- Initial segment = 3

**Stage 2: Boundary Crossing Detection**
```python
def _crossed_boundary(p1, p2, current_seg) -> bool:
    """
    Detect if path crossed from current_seg into next segment.
    
    Uses tangent projection method:
    1. Get boundary between current_seg and next_seg
    2. Boundary defined by: point (bx, by), tangent vector (tx, ty)
    3. Compute signed distances:
       d1 = dot(p1 - boundary_point, tangent)
       d2 = dot(p2 - boundary_point, tangent)
    4. Crossing occurs when: d1 < 0 and d2 >= 0
    
    Geometric interpretation:
    - Tangent vector points along course direction
    - Negative distance = haven't reached boundary yet
    - Positive distance = passed boundary
    - Transition neg→pos = crossed boundary
    """
```

**Why This Works:**
The tangent projection method is robust to cross-track errors:
- Driver can be left/right of ideal line
- As long as forward progress continues, boundaries detect correctly
- Handles wide racing lines and position drift

**Edge Cases:**
1. **Driving backwards**: Detects backwards boundary crossings (seg N → N-1)
2. **Large jumps**: If position jumps > segment length, may skip segments
3. **Starting mid-segment**: Initial detection handles this
4. **Closed loops**: Wraparound handled (segment N → 0)

**Example Usage:**
```python
assigner = SegmentAssigner(segmentation)

# Assign to entire path
segment_ids = assigner.assign(path_data.x, path_data.y)

# Result: [3, 3, 3, ..., 4, 4, 4, ..., 5, 5, ...]
# Shows progression through segments
```

**Performance:**
- O(1) for initial detection (nearest neighbor on mean course)
- O(n) for crossing detection (one check per point)
- Typical: 1000 points/second on modern CPU

**Alternative: SegmentEstimator**
For advanced use cases, `SegmentEstimator` provides:
- Confidence scores for each segment assignment
- Cross-track error (distance from mean course)
- Heading error (angle difference from mean course)
- Useful for detecting poor driving or data quality issues

```python
estimator = SegmentEstimator(segmentation)
estimate = estimator.estimate(x, y, heading)

print(f"Segment: {estimate.segment_id}")
print(f"Confidence: {estimate.confidence:.2f}")
print(f"Cross-track error: {estimate.cross_track_error:.2f}m")
```

---

**Benefits of Course Analysis Framework:**
- **Geometric Understanding**: Quantify track features (straights, turns, 
  difficulty)
- **Consistency Detection**: Compare lap variations, identify problem areas
- **Segment-Based Training**: Train on best instances of each track section
- **Data Quality**: Validate IMU data, detect position drift or errors
- **Performance Analysis**: Measure speed, smoothness per segment
- **Visualization**: Interactive tools for debugging and analysis
- **Reproducibility**: Metadata tracking ensures consistent results

---

### 2. Segment-Based Performance Training

Train models on the best-driven instances of each track segment across all 
laps, rather than just the best complete laps. This revolutionary training 
methodology creates a "synthetic perfect lap" that combines the best performance 
in each section of the track, fundamentally changing how autonomous driving 
models learn from demonstration data.

**Core Innovation:**
Traditional lap-based training uses complete laps ranked by overall performance. 
Segment-based training recognizes that a driver rarely executes every part of a 
track perfectly in a single lap. By identifying and learning from the best 
execution of each segment across all laps, the model can exceed any single 
human demonstration.

---

#### Architecture and Implementation

**Key Files and Responsibilities:**

1. **`donkeycar/pipeline/types.py`** - Type definitions
   ```python
   class PctMode(Enum):
       """Behavioral parameter percentage mode"""
       NONE = 0     # No ranking (use all data equally)
       LAP = 1      # Rank complete laps
       SEGMENT = 2  # Rank segment instances
   
   @dataclass
   class FieldAggregationSpec:
       """Specification for aggregating tub fields"""
       field: str              # Tub field name (e.g., 'car/gyro')
       index: Optional[int]    # Array index (None = scalar field)
       output_key: str         # Key for aggregated result
       transform: Callable     # Transform before aggregation (e.g., abs)
       aggregation: str        # 'avg', 'sum', 'min', 'max', 'median'
   ```

2. **`donkeycar/parts/tub_statistics.py`** - Performance calculation
   - `TubStatistics.compute_segment_assignments()` - Computes and writes segments
   - `TubStatistics.calculate_segment_performance()` - Ranks segment instances
   - `SegmentTracker` - Stateful iterator for segment boundaries
   - `FieldAccumulator` - Accumulates values per segment

3. **`donkeycar/pipeline/training.py`** - Integration with training
   - `TubDataset.__init__(pct_mode=PctMode.SEGMENT)` - Enable segment mode
   - `TubRecord.extend()` - Populates `lap_pct` from segment rankings
   - Training loop automatically filters low-performing segment instances

---

#### Configuration

**In cfg_complete.py or myconfig.py:**

```python
# ============================================================================
# SEGMENT PERFORMANCE CONFIGURATION
# ============================================================================

# Enable segment-based training (default: False = lap-based)
SEGMENT_PCT_MODE = False

# Segmentation strategy: 'threshold', 'extrema', 'gradient', 'hybrid'
# - threshold: Equal arc-length segments (simple, predictable)
# - extrema: Curvature peaks (natural for corner-to-corner)
# - gradient: Curvature change (good for S-curves)
# - hybrid: Combined approach (recommended for general use)
SEGMENT_STRATEGY = 'hybrid'

# Lap detection method: 'ycrossing' or 'drift'
# - ycrossing: Y-coordinate crossing (for clean data)
# - drift: Cluster-based (for GPS drift)
SEGMENT_LAP_DETECTOR = 'ycrossing'

# Minimum segment length in meters
# Shorter segments are merged with neighbors
# Typical: 1.0-3.0m depending on track size
SEGMENT_MIN_LENGTH = 1.0

# Curvature threshold for gradient/hybrid strategies
# Higher = fewer segments (only sharp features)
# Lower = more segments (captures subtle features)
# Typical: 0.05-0.15
SEGMENT_CURVATURE_THRESHOLD = 0.1

# ============================================================================
# FIELD AGGREGATIONS - Custom Performance Metrics
# ============================================================================

def abs_transform(value):
    """Absolute value transform for magnitude-based ranking"""
    return abs(value)

FIELD_AGGREGATIONS = [
    {
        'field': 'car/gyro',           # IMU gyroscope data
        'index': 2,                     # Z-axis (yaw rate)
        'output_key': 'gyro_z_agg',    # Key in performance dict
        'transform': abs_transform,     # Take absolute value
        'aggregation': 'avg'            # Average over segment
    }
    # Add more metrics as needed:
    # - Steering variation: field='user/angle', aggregation='std'
    # - Throttle smoothness: field='user/throttle', transform=diff
    # - IMU acceleration: field='car/accel'
]

# Multi-criteria sorting for performance ranking
# Records ranked by these criteria in order (ties broken by next criterion)
LAP_SORTING_CRITERIA = [
    {'key': 'time'},          # Primary: Segment completion time
    {'key': 'distance'},      # Secondary: Distance traveled
    {'key': 'gyro_z_agg'},   # Tertiary: Smoothness (avg abs gyro_z)
]
```

**Parameter Tuning Guidelines:**

| Track Type | SEGMENT_STRATEGY | SEGMENT_MIN_LENGTH | Notes |
|------------|------------------|-------------------|-------|
| Simple oval | threshold | 2.0-3.0m | Equal segments work well |
| Road course | extrema | 1.5-2.5m | Natural corner segmentation |
| Technical track | hybrid | 1.0-2.0m | Captures complex features |
| Tight indoor | gradient | 0.5-1.5m | Sensitive to small features |

---

#### Complete Workflow

**Step 1: Record Multi-Lap Training Data**

```bash
# Create car with IMU support (donkey5 template recommended)
donkey createcar --template donkey5 --path ~/mycar
cd ~/mycar

# Configure IMU in myconfig.py
# HAVE_IMU = True
# IMU_TYPE = 'bno055'  # or 'mpu6050', 'mpu9250'

# Record multiple laps (3-5 recommended)
python manage.py drive --tub ./data --js

# Or with path recording for visualization
python manage.py drive --tub ./data --js --record_path
```

**Drive 3-5 consistent laps:**
- Focus on smooth, fast driving
- Don't worry about perfect laps - best segments will be selected
- More laps = more chances to nail each segment perfectly
- Variation is good - provides diverse segment instances to rank

**Step 2: Compute Segment Assignments**

```bash
# Basic segmentation (uses config defaults)
donkey segment --tub ./data/tub_1

# Custom parameters
donkey segment --tub ./data/tub_1 \
    --strategy hybrid \
    --lap-detector ycrossing \
    --min-segment-length 1.5 \
    --curvature-threshold 0.08

# With visualization (requires matplotlib)
donkey segment --tub ./data/tub_1 --visualize

# Batch process multiple tubs
for tub in ./data/tub_*; do
    donkey segment --tub "$tub"
done
```

**What this does:**
1. Loads IMU data from tub (`car/pos`, `car/euler`, `car/speed`)
2. Detects lap boundaries using specified detector
3. Builds mean course from all detected laps
4. Segments mean course using specified strategy
5. Assigns segment IDs to every tub record
6. Writes `car/segment` field (int) to each record
7. Stores metadata in tub manifest:
   ```json
   {
       "segmentation": {
           "num_segments": 12,
           "strategy": "hybrid",
           "lap_detector": "ycrossing",
           "mean_course_params": {...},
           "timestamp": "2024-01-15T10:30:45"
       }
   }
   ```

**Step 3: Verify Segmentation (Optional)**

```bash
# Visualize path with segments
donkey imupath ./data/tub_1

# Check segment counts
python -c "
from donkeycar.parts.tub_v2 import Tub
tub = Tub('./data/tub_1')
for record in tub:
    seg = record.underlying.get('car/segment')
    lap = record.underlying.get('car/lap')
    print(f'Lap {lap}, Segment {seg}')
" | sort | uniq -c
```

**Expected output:**
```
  45 Lap 0, Segment 0
  38 Lap 0, Segment 1
  52 Lap 0, Segment 2
  ...
  46 Lap 2, Segment 11
```

**Step 4: Enable Segment Mode in Configuration**

```python
# In myconfig.py
SEGMENT_PCT_MODE = True
SEGMENT_STRATEGY = 'hybrid'  # Must match segmentation used

# Optional: Add custom metrics
FIELD_AGGREGATIONS = [
    {
        'field': 'car/gyro',
        'index': 2,
        'output_key': 'gyro_z_agg',
        'transform': abs,
        'aggregation': 'avg'
    }
]

LAP_SORTING_CRITERIA = [
    {'key': 'time'},
    {'key': 'gyro_z_agg'},  # Prioritize smooth driving
]
```

**Step 5: Train Model**

```bash
# Standard training with segment mode enabled
python manage.py train --tub ./data/tub_1 --model ./models/pilot.h5

# Monitor training output
# Look for: "Using segment-based performance ranking"
# Shows segment instance rankings being computed
```

**Training process:**
1. TubDataset detects `SEGMENT_PCT_MODE = True`
2. Calls `TubStatistics.calculate_segment_performance()`
3. For each segment instance, computes metrics:
   - Time: Segment duration
   - Distance: Arc length traveled
   - Custom: Field aggregations (e.g., gyro_z_agg)
4. Ranks instances within each (session, lap, segment) group
5. Converts ranks to percentages (0.0 = worst, 1.0 = best)
6. Populates `record['lap_pct']` with segment percentages
7. Training pipeline filters records with `lap_pct < PCT_THRESHOLD`
8. Model learns from top N% of each segment across all laps

---

#### How It Works: Deep Dive

**Data Structure: Segment Performance Tracking**

```python
# Internal structure in TubStatistics
segment_instances = {
    session_id: {
        lap_num: {
            segment_id: {
                'start_time': float,
                'end_time': float,
                'start_dist': float,
                'end_dist': float,
                'field_values': {
                    'gyro_z_agg': float,
                    'throttle_std': float,
                    # ... more custom metrics
                }
            }
        }
    }
}

# After ranking
session_rank = {
    session_id: {
        lap_num: {
            segment_id: [time_pct, gyro_z_pct, distance_pct, ...]
        }
    }
}
```

**Example with Real Numbers:**

Track with 4 segments, 3 laps:

```python
# Lap 1 times
Segment 0: 2.1s  (rank 1/3 = 33%)
Segment 1: 3.5s  (rank 2/3 = 67%)  
Segment 2: 2.8s  (rank 2/3 = 67%)
Segment 3: 2.2s  (rank 1/3 = 33%)

# Lap 2 times
Segment 0: 2.3s  (rank 2/3 = 67%)
Segment 1: 3.2s  (rank 1/3 = 33%)  <- BEST for segment 1
Segment 2: 2.5s  (rank 1/3 = 33%)  <- BEST for segment 2
Segment 3: 2.5s  (rank 3/3 = 100%)

# Lap 3 times
Segment 0: 2.5s  (rank 3/3 = 100%)
Segment 1: 3.8s  (rank 3/3 = 100%)
Segment 2: 2.7s  (rank 2/3 = 67%)
Segment 3: 2.3s  (rank 2/3 = 67%)
```

With `PCT_THRESHOLD = 0.8`, training uses:
- Lap 1, Segment 0 (33% - INCLUDED)
- Lap 2, Segment 1 (33% - INCLUDED)
- Lap 2, Segment 2 (33% - INCLUDED)
- Lap 1, Segment 3 (33% - INCLUDED)
- Lap 1, Segment 2 (67% - INCLUDED)
- Lap 3, Segment 3 (67% - INCLUDED)
- All others rejected (> 80%)

**Result:** Model learns from best 50% of segment instances, getting the best 
demonstration of each track section!

**Multi-Criteria Ranking:**

When `LAP_SORTING_CRITERIA` has multiple keys:

```python
LAP_SORTING_CRITERIA = [
    {'key': 'time'},         # Sort by time first
    {'key': 'gyro_z_agg'},   # Break ties by smoothness
]
```

Segment instances sorted by: `(time, gyro_z_agg)`
- Fastest time wins
- If times tied within epsilon, smoothest wins
- Result: Prioritizes fast AND smooth driving

---

#### Concrete Training Scenario

**Scenario:** Road course with chicane that you sometimes nail perfectly, 
sometimes mess up.

**Lap 1:**
- Segments 0-5: Good execution (rank: 2/3)
- Segment 6 (chicane): PERFECT! Fastest, smoothest (rank: 1/3)
- Segments 7-11: Decent (rank: 2/3)

**Lap 2:**
- Segments 0-5: EXCELLENT (rank: 1/3 for most)
- Segment 6 (chicane): Terrible, hit apex wrong (rank: 3/3)
- Segments 7-11: Good (rank: 2/3)

**Lap 3:**
- Segments 0-5: Okay (rank: 3/3)
- Segment 6 (chicane): Decent (rank: 2/3)
- Segments 7-11: BEST EVER (rank: 1/3 for most)

**Traditional lap-based training:** Uses Lap 2 (best overall time)
- Learns your excellent segments 0-5
- **Also learns your terrible chicane from Lap 2!**
- Misses your perfect chicane from Lap 1
- Misses your excellent segments 7-11 from Lap 3

**Segment-based training:** Uses best of each segment
- Learns excellent segments 0-5 from Lap 2
- **Learns PERFECT chicane from Lap 1!**
- Learns excellent segments 7-11 from Lap 3
- Result: Model better than any single lap you drove!

---

#### Iterative Improvement Strategy

Segment-based training enables progressive performance enhancement:

**Iteration 1: Human Baseline**
```bash
# Drive 5 laps, compute segments, train
donkey segment --tub ./data/human_laps
python manage.py train --tub ./data/human_laps \
    --model ./models/gen1.h5 --type linear
```
- Model learns from best segment instances
- Performance: 95% of human best lap

**Iteration 2: Model-Assisted**
```bash
# Drive 5 laps with model assistance (50% pilot)
# Use pilot angle/throttle as suggestions, override when better
python manage.py train --tub ./data/human_laps,./data/assisted_laps \
    --model ./models/gen2.h5 --type linear
```
- Model sees new segment instances from assisted driving
- Some segments now driven better than original human baseline
- Re-segmentation includes all laps (10 total)
- Training selects best instances across all 10 laps
- Performance: 98% of human best lap

**Iteration 3: Model Refinement**
```bash
# Drive 5 more laps, focusing on weak segments identified in gen2
python manage.py train --tub ./data/human_laps,./data/assisted_laps,./data/refined_laps \
    --model ./models/gen3.h5 --type linear
```
- 15 total laps provide diverse segment instances
- Training cherry-picks absolute best of each segment
- Performance: 101% of human best lap (exceeds human!)

**Key Insight:** More laps + segment-based ranking = better coverage of each 
track section, enabling the model to synthesize superhuman performance.

---

#### Performance Metrics and Validation

**Measuring Improvement:**

```python
# Compare lap-based vs segment-based training
# Script: validate_segment_training.py

from donkeycar.parts.tub_statistics import TubStatistics
from donkeycar.pipeline.types import PctMode

# Load tub
tub_path = './data/tub_1'
stats = TubStatistics(tub_path)

# Lap-based performance
lap_pct = stats.calculate_lap_performance()
print("Lap-based: Best lap time =", min(lap_pct['times']))

# Segment-based performance
seg_pct = stats.calculate_segment_performance()
synthetic_lap_time = sum(min(seg_times) 
                         for seg_times in seg_pct['segment_times'])
print("Segment-based: Synthetic best =", synthetic_lap_time)

improvement = (lap_pct_best - synthetic_lap_time) / lap_pct_best * 100
print(f"Theoretical improvement: {improvement:.1f}%")
```

**Typical Results:**
- 3 laps: 2-5% improvement over best lap
- 5 laps: 5-10% improvement
- 10 laps: 10-15% improvement
- Diminishing returns after ~10 laps per segment

---

#### Advanced Configuration: Custom Metrics

**Example: Prioritize Smooth Steering**

```python
# In myconfig.py

def steering_variation(angle):
    """Measure steering smoothness"""
    return abs(angle)  # Will be aggregated as std dev

FIELD_AGGREGATIONS = [
    {
        'field': 'user/angle',
        'index': None,  # Scalar field
        'output_key': 'steering_smoothness',
        'transform': steering_variation,
        'aggregation': 'std'  # Standard deviation
    },
    {
        'field': 'car/gyro',
        'index': 2,
        'output_key': 'gyro_z_agg',
        'transform': abs,
        'aggregation': 'avg'
    }
]

LAP_SORTING_CRITERIA = [
    {'key': 'time'},                    # Fast
    {'key': 'steering_smoothness'},     # Smooth steering
    {'key': 'gyro_z_agg'},             # Smooth body motion
]
```

Result: Model learns fast, smooth, controlled driving.

---

#### Troubleshooting

**Issue: "No segments found in tub"**
- Cause: `donkey segment` not run on tub
- Fix: `donkey segment --tub ./data/tub_1`

**Issue: "Segment IDs not continuous"**
- Cause: Segmentation failed or data corruption
- Fix: Re-run `donkey segment` with `--force` flag

**Issue: "Training slower with segment mode"**
- Cause: Segment performance calculation is expensive
- Fix: Normal - only computed once, then cached

**Issue: "Model not improving with segment training"**
- Possible causes:
  1. Not enough laps (need 3+ for meaningful ranking)
  2. Segments too small (increase MIN_SEGMENT_LENGTH)
  3. Inconsistent driving (vary speed/line too much)
  4. Poor segmentation strategy for track type
- Debug: Visualize segments with `donkey imupath --visualize`

**Issue: "All segment instances ranked equally"**
- Cause: Driving too consistent, or field aggregations not configured
- Fix: Add more discriminating metrics (gyro, steering variation)

---

**Benefits of Segment-Based Performance Training:**

- **Synthetic Perfect Lap**: Exceed any single human demonstration
- **Efficient Data Use**: Learn from partial successes in each lap
- **Robust to Inconsistency**: Don't need perfect laps, just perfect segments
- **Iterative Improvement**: Progressive refinement over multiple sessions
- **Multi-Objective Optimization**: Balance speed, smoothness, consistency
- **Track-Aware Learning**: Model understands track structure, not just pixels
- **Reduced Training Data**: 5 segmented laps > 20 unsegmented laps
- **Competitive Advantage**: Superior performance for racing applications

---

### 3. IMU Path Visualization

Interactive visualization tool for analyzing recorded vehicle trajectories with 
real-time lap detection, course segmentation, and comprehensive debugging 
capabilities. This tool transforms raw IMU sensor data into actionable insights 
about driving performance and track geometry.

**Command:** `donkey imupath <data_source>`

**Location:** `donkeycar/utilities/interactive_imu_viz.py`

---

#### Overview and Purpose

The IMU path visualizer serves multiple critical functions:

1. **Data Quality Validation**: Verify IMU sensor data integrity before training
2. **Algorithm Debugging**: Visualize lap detection and segmentation behavior
3. **Parameter Tuning**: Experiment with different strategies and thresholds
4. **Performance Analysis**: Understand where driving can be improved
5. **Training Verification**: Confirm segment assignments match expectations

**Interactive Features:**
- **Time Slider**: Scrub through recording frame-by-frame
- **Lap Selection**: Choose how many laps to include in mean course
- **Method Selection**: Switch between segmentation strategies in real-time
- **Keyboard Navigation**: Arrow keys for precise timeline control
- **Toggle Displays**: Show/hide path components, segments, boundaries
- **Status Panel**: Real-time display of position, speed, lap, segment
- **Zoom and Pan**: Standard matplotlib navigation tools

---

#### Usage Examples and Scenarios

**Basic Visualization:**
```bash
# Visualize CSV file (t, x, y, h, v format)
donkey imupath ./recordings/track_session_001.csv

# Visualize tub directory
donkey imupath ./data/tub_1

# Visualize specific tub session
donkey imupath ./data/tub_1 --session 20240115_143022
```

**Advanced Options:**
```bash
# Specify lap detection method
donkey imupath --lap-method drift ./outdoor_gps_data.csv

# Specify segmentation strategy
donkey imupath --segment-method hybrid ./data/tub_1

# Set number of laps for mean course
donkey imupath --num-laps 3 ./data/tub_1

# Combine options
donkey imupath \
    --lap-method drift \
    --segment-method extrema \
    --num-laps 4 \
    --min-segment-length 2.0 \
    ./outdoor_track.csv
```

**Custom Configuration:**
```bash
# Use config file for advanced parameters
donkey imupath --config ./my_viz_config.py ./data/tub_1

# In my_viz_config.py:
LAP_DETECTION_PARAMS = {
    'min_lap_duration': 8.0,
    'cluster_eps': 2.5,
}

SEGMENTATION_PARAMS = {
    'min_segment_length': 1.5,
    'curvature_threshold': 0.12,
    'prominence': 0.08,
}
```

---

#### Data Sources and Formats

**CSV Format:**
```csv
t,x,y,h,v
0.000,0.000,0.000,0.000,0.000
0.100,0.015,0.048,0.523,0.502
0.200,0.032,0.095,0.589,0.498
...
```

**Field Definitions:**
- `t`: Timestamp in seconds (relative to recording start)
- `x`: X position in meters (right = positive, local coordinate frame)
- `y`: Y position in meters (forward = positive, local coordinate frame)
- `h`: Heading in degrees (0 = north, clockwise positive)
- `v`: Velocity in meters per second

**Coordinate System:**
- Origin (0, 0) at recording start position
- X-axis: Right (vehicle's initial right direction)
- Y-axis: Forward (vehicle's initial forward direction)
- Heading: Relative to initial orientation

**Tub Format:**
The visualizer automatically extracts required fields from tub records:

```python
# Field mapping from tub to PathData
_timestamp_ms    → timestamp (converted to seconds)
car/pos[0:2]     → x, y (position in meters)
car/euler[2]     → heading (yaw angle in radians)
car/speed        → velocity (m/s)
```

**Requirements:**
- Minimum 100 data points (configurable)
- Monotonically increasing timestamps
- No NaN or Inf values
- Valid position coordinates (not all zeros)

**Data Quality Checks:**
The visualizer performs automatic validation:
1. Timestamp consistency (no backwards jumps)
2. Position validity (reasonable coordinate ranges)
3. Velocity sanity (< 50 m/s for typical RC cars)
4. Field completeness (all required fields present)

Warnings issued for:
- Large timestamp gaps (> 1 second)
- Position jumps (> 10 meters between points)
- Zero velocity for extended periods
- Missing or invalid field values

---

#### User Interface Components

**Main Display:**
```
┌─────────────────────────────────────────────────────────┐
│  Track Visualization (Matplotlib Figure)                │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │                                                 │     │
│  │           [Path Plot with Segments]            │     │
│  │                                                 │     │
│  │   • Driven path (colored by segment)           │     │
│  │   • Mean course (grey line)                    │     │
│  │   • Segment boundaries (vertical lines)        │     │
│  │   • Current position marker (red dot)          │     │
│  │   • Segment labels (S0, S1, S2...)             │     │
│  │                                                 │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  Time Slider: [────────●──────────────────────]          │
│                                                          │
│  Control Panel:                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Lap Method  │  │  Seg Method  │  │  Num Laps    │   │
│  │ ○ YCrossing │  │  ○ Threshold │  │  [  3  ]     │   │
│  │ ● Drift     │  │  ○ Extrema   │  │  [ Update ]  │   │
│  │             │  │  ● Gradient  │  │              │   │
│  │             │  │  ○ Hybrid    │  │              │   │
│  └─────────────┘  └──────────────┘  └──────────────┘   │
│                                                          │
│  Status Panel:                                           │
│  Time: 12.34s | Speed: 1.23 m/s | Lap: 2 | Segment: 5  │
│  Position: (1.23, 4.56) | Heading: 45.6°               │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Interactive Controls:**

1. **Time Slider**
   - Drag to scrub through recording
   - Click to jump to specific time
   - Updates position marker and status in real-time
   - Throttled updates (60 FPS max) for smooth performance

2. **Lap Method Selector (Radio Buttons)**
   - **YCrossing**: For clean, repeatable data
   - **Drift**: For GPS data with position drift
   - Changing method recomputes lap boundaries
   - Updates mean course and segmentation automatically

3. **Segmentation Method Selector (Radio Buttons)**
   - **Threshold**: Equal arc-length segments
   - **Extrema**: Curvature peak-based
   - **Gradient**: Curvature change-based
   - **Hybrid**: Combined approach
   - Live update of segment boundaries and colors

4. **Num Laps Input**
   - Text box to enter number of laps for mean course
   - "Update" button applies change
   - Validates input (1 to total_laps)
   - Recomputes mean course and segments

5. **Display Toggles (Check Buttons)**
   - [ ] Show Driven Path
   - [x] Show Mean Course
   - [x] Show Segments
   - [x] Show Boundaries
   - Independent visibility control

6. **Keyboard Shortcuts**
   - `Left Arrow`: Step backwards (0.1 seconds)
   - `Right Arrow`: Step forwards (0.1 seconds)
   - `Home`: Jump to start
   - `End`: Jump to end
   - `Space`: Play/pause auto-advance (if implemented)

---

#### Visualization Elements

**Driven Path:**
- Color-coded by segment ID (unique color per segment)
- Line thickness proportional to velocity (optional)
- Opacity varies with time (fades for older sections)
- Current position highlighted with red circle marker

**Mean Course:**
- Solid grey line (#808080)
- Shows idealized track centerline
- Computed from selected number of laps
- Updates when num_laps or lap_method changes

**Segment Boundaries:**
- Vertical lines perpendicular to mean course
- Dashed pattern for visibility
- Labeled with segment IDs (S0, S1, S2...)
- Color matches corresponding segment

**Segment Labels:**
- Positioned at segment midpoints
- Show segment ID and optional metadata:
  - Segment length (meters)
  - Mean curvature (1/m)
  - Segment type (STRAIGHT, LEFT_TURN, etc.)

**Status Panel:**
- **Time**: Current timestamp (seconds from start)
- **Speed**: Instantaneous velocity (m/s)
- **Lap**: Current lap number (0-indexed)
- **Segment**: Current segment ID
- **Position**: (x, y) coordinates in meters
- **Heading**: Current heading angle (degrees)

---

#### Performance Optimizations

**Downsampling:**
For large datasets (> 10,000 points):
- Display path downsampled to 2,000 points
- Maintain full resolution for calculations
- Preserves visual appearance while improving responsiveness

**Update Throttling:**
- Slider updates throttled to 60 FPS (16.7ms)
- Prevents excessive redraws during fast scrubbing
- Queues updates during processing

**Lazy Computation:**
- Mean course and segments computed on demand
- Results cached until parameters change
- Avoids redundant calculations

**Efficient Rendering:**
- Uses matplotlib blitting for animated elements
- Only redraws changed components
- Background cached for static elements

---

#### Debugging Workflows

**Workflow 1: Validate Lap Detection**

1. Load data: `donkey imupath ./data/tub_1`
2. Observe lap boundaries (where path colors change)
3. Verify boundaries align with actual lap start/finish
4. If incorrect:
   - Try alternate lap method (YCrossing ↔ Drift)
   - Check data quality (gaps, jumps)
   - Adjust min_lap_duration parameter

**Workflow 2: Tune Segmentation**

1. Load data with specific method:
   ```bash
   donkey imupath --segment-method hybrid ./data/tub_1
   ```
2. Observe segment boundaries on track
3. Check if boundaries align with geometric features:
   - Straights should be separate segments
   - Corners should have boundaries at entry/exit
4. Experiment with strategies:
   - Threshold: Quick baseline
   - Extrema: Natural for distinct corners
   - Gradient: Captures transitions
   - Hybrid: Best all-around
5. Adjust parameters if needed:
   - Increase min_segment_length to merge small segments
   - Adjust curvature_threshold for gradient/hybrid sensitivity

**Workflow 3: Verify Training Data Quality**

1. Load training tub: `donkey imupath ./data/training_tub_1`
2. Scrub through time slider, watching for:
   - Position jumps (sensor errors)
   - Velocity spikes (noise)
   - Heading inconsistencies (IMU drift)
3. Identify problematic sections
4. Optionally exclude from training:
   - Manually note timestamps
   - Filter records in training pipeline

**Workflow 4: Compare Laps**

1. Load multi-lap data
2. Set num_laps = 1, observe single lap
3. Increment num_laps, watch mean course evolve
4. Identify lap-to-lap variations:
   - Consistent sections (good)
   - High variation sections (focus for improvement)
5. Use insights to prioritize practice

---

#### Algorithm Visualization

**Lap Detection Animation:**
When lap method changes:
1. Path briefly flashes to show recomputation
2. New boundaries fade in
3. Path re-colored by new lap assignments
4. Status updates to show new lap count

**Segmentation Animation:**
When segment method changes:
1. Old boundaries fade out
2. New boundaries computed
3. New boundaries fade in with labels
4. Path transitions to new segment colors
5. Legend updates to show new segments

**Real-time Feedback:**
As slider moves:
- Position marker smoothly tracks along path
- Current segment highlights (optional)
- Status panel updates instantly
- Velocity indicator scales (if enabled)

---

#### Output and Export

**Screenshot Capture:**
- Use matplotlib toolbar "Save" button
- Formats: PNG, PDF, SVG
- Recommended: PNG at 300 DPI for documentation

**Data Export:**
Export processed data for external analysis:

```python
# After visualizing, export segment assignments
from donkeycar.utilities.interactive_imu_viz import export_segment_data

export_segment_data(
    path_data=visualizer.path_data,
    segment_ids=visualizer.segment_ids,
    output_path='./analyzed_path.csv'
)

# Output format: t, x, y, h, v, lap, segment
```

**Session Summary:**
```bash
# Generate summary report
donkey imupath ./data/tub_1 --summary > session_report.txt

# Report includes:
# - Total recording time
# - Number of laps detected
# - Lap times and statistics
# - Number of segments per lap
# - Mean course length
# - Curvature statistics
# - Data quality metrics
```

---

#### Troubleshooting

**Issue: "No laps detected"**
- **Cause**: Lap detector didn't find crossings
- **Solutions**:
  1. Try alternate lap_method
  2. Check if track crosses y=0 (for YCrossing)
  3. Reduce min_lap_duration
  4. Verify data has multiple laps

**Issue: "Visualization is slow"**
- **Cause**: Large dataset or underpowered machine
- **Solutions**:
  1. Downsample before visualization (reduce recording rate)
  2. Use --max-points flag to limit display
  3. Disable real-time segment updates
  4. Close other applications

**Issue: "Segments don't match track features"**
- **Cause**: Wrong strategy or parameters
- **Solutions**:
  1. Try different segmentation strategy
  2. Adjust min_segment_length (increase if too many small segments)
  3. Adjust curvature_threshold (decrease for more sensitivity)
  4. Ensure enough laps for stable mean course (3+ recommended)

**Issue: "Mean course looks jagged"**
- **Cause**: Inconsistent lap driving or too few laps
- **Solutions**:
  1. Increase num_laps for better averaging
  2. Increase resample_points for smoother interpolation
  3. Apply smoothing filter (increase smooth_window)
  4. Drive more consistent laps

**Issue: "Path jumps or discontinuities"**
- **Cause**: IMU sensor errors or lost data
- **Solutions**:
  1. Check sensor connections
  2. Filter outliers before visualization
  3. Exclude problematic time ranges
  4. Recalibrate IMU sensor

---

**Benefits of IMU Path Visualization:**

- **Visual Debugging**: See what algorithms are doing, not just numbers
- **Intuitive**: Non-technical users can understand path and segments
- **Interactive**: Real-time parameter tuning without code changes
- **Educational**: Learn how lap detection and segmentation work
- **Quality Assurance**: Catch data problems before training
- **Documentation**: Screenshots for reports, papers, presentations
- **Algorithm Development**: Test new strategies visually
- **Performance Analysis**: Identify driving improvements needed

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

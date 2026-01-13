# PyTorch and Python 3.12 Migration Evaluation

**Date:** January 2026  
**Current State:** Python 3.11.0-3.12 (exclusive), TensorFlow 2.15, PyTorch 2.1 (optional)  
**Evaluation:** Feasibility of migrating to PyTorch as primary framework and Python 3.12

---

## Executive Summary

This document evaluates the feasibility of updating donkeycar to use PyTorch as the primary ML framework and Python 3.12. The evaluation concludes that:

✅ **Python 3.12 migration is FEASIBLE** with dependency updates  
⚠️ **PyTorch as primary framework is POSSIBLE but COMPLEX** due to extensive TensorFlow integration  
⚠️ **Platform-specific challenges exist** for Raspberry Pi and Jetson deployments

**Recommendation:** Incremental approach - Update to Python 3.12 first with TensorFlow 2.16+, then gradually expand PyTorch support while maintaining TensorFlow compatibility.

---

## Current State Analysis

### Python Version Requirements
- **Current:** `>=3.11.0,<3.12` (setup.cfg line 26)
- **CI/CD:** GitHub Actions configured for Python 3.11 (python-package-conda.yml)
- **Classifier:** `Programming Language :: Python :: 3.11` (setup.cfg line 20)

### ML Framework Usage

#### TensorFlow/Keras (Primary Framework)
**Files with TensorFlow imports:** 8 core files
- `donkeycar/parts/keras.py` (1,183 lines) - Main pilot implementations
- `donkeycar/parts/keras_2.py` (1,021 lines) - Alternative Keras implementations
- `donkeycar/pipeline/training.py` (235 lines) - Training pipeline
- `donkeycar/parts/interpreter.py` - TFLite, TensorRT, Keras interpreters
- `donkeycar/parts/salient.py` - Saliency visualization
- `donkeycar/management/makemovie.py` - Video generation
- Multiple template files (complete.py, simulator.py, etc.)

**TensorFlow Version:** 2.15.* across all platforms
- **PC/macOS:** `tensorflow==2.15.*`
- **Raspberry Pi:** `tensorflow-aarch64==2.15.*`
- **Jetson Nano:** No TensorFlow specified (uses system install)

**Key Dependencies:**
- Uses Keras API extensively with `tf.keras.layers`, `tf.keras.models`
- TensorFlow Lite conversion for edge deployment
- TensorRT support for GPU inference
- Direct TensorFlow Python APIs (`tensorflow.python.keras.models`)

#### PyTorch (Optional Framework)
**Files with PyTorch imports:** 4 files in `donkeycar/parts/pytorch/`
- `ResNet18.py` - Pre-trained ResNet18 model
- `torch_data.py` - Data loading and transforms
- `torch_train.py` - Training pipeline
- `torch_utils.py` - Utilities

**PyTorch Version:** 2.1.* (optional install)
- Includes: `pytorch-lightning`, `torchvision`, `torchaudio`, `fastai`
- Status: Optional extra (`pip install -e .[torch]`)
- Integration: Separate from main training pipeline

**Coverage:** Limited to experimental/alternative training path

---

## Python 3.12 Compatibility Analysis

### Core Framework Compatibility

#### TensorFlow
| Version | Python 3.12 | Status | Notes |
|---------|-------------|--------|-------|
| 2.15.x | ❌ No | Current | Supports only up to Python 3.11 |
| 2.16.x | ✅ Yes | Required | First version with Python 3.12 support |
| 2.17.x | ✅ Yes | Alternative | Latest stable as of 2025 |

**Critical Issues with TensorFlow 2.15 + Python 3.12:**
- No pre-built wheels available for Python 3.12
- `pip install tensorflow==2.15.*` fails on Python 3.12
- Building from source not officially supported
- Backporting Python 3.12 support to 2.15 is not planned

**TensorFlow 2.16+ Considerations:**
- ✅ Official Python 3.12 support
- ⚠️ GPU detection issues reported (especially 2.16-2.17)
- ⚠️ Defaults to Keras 3 (potential breaking changes)
- ⚠️ Drops `tf.estimator` API (not used in donkeycar)
- ⚠️ Some stability issues with Python 3.12 reported by users
- ✅ Works on Apple Silicon with standard `pip install tensorflow`

#### PyTorch
| Version | Python 3.12 | Status | Notes |
|---------|-------------|--------|-------|
| 2.1.x | ❌ No | Current | Supports only up to Python 3.11 |
| 2.2.x | ✅ Yes | Minimum | First version with Python 3.12 support |
| 2.4.x | ✅ Yes | Recommended | Stable Python 3.12 support |

**Critical Issues with PyTorch 2.1 + Python 3.12:**
- No pre-built wheels for Python 3.12
- Installation fails with dependency errors
- Upgrade to 2.2+ required for Python 3.12

**PyTorch 2.4+ Benefits:**
- ✅ Official, stable Python 3.12 support
- ✅ Improved `torch.compile` performance
- ✅ Better CUDA integration
- ✅ Native Windows ARM64 wheels

#### PyTorch Lightning
- **Current:** No version specified (uses latest)
- **Python 3.12:** Probable support with PyTorch 2.2+ but not explicitly documented
- **Recommendation:** Specify version `>=2.5.0` for best compatibility

### Platform-Specific Compatibility

#### Raspberry Pi (ARM64/aarch64)
**Current State:**
- `tensorflow-aarch64==2.15.*`
- `flatbuffers==24.3.*`
- Python 3.11 required

**Python 3.12 Implications:**
- ✅ Python 3.12 works on ARM64 architecture
- ⚠️ `tensorflow-aarch64` 2.16+ required for Python 3.12
- ⚠️ May need to rebuild native libraries
- ✅ PyTorch official aarch64 wheels available for Python 3.12
- ⚠️ Piwheels only supports 32-bit (armhf), not 64-bit OS
- ✅ Can use PyPI/conda-forge for 64-bit ARM packages

#### Jetson Nano
**Current State:**
- No TensorFlow in requirements (uses NVIDIA-provided)
- `numpy==1.23.*`, `matplotlib==3.7.*`, `pandas==2.0.*`
- Older locked versions for compatibility

**Python 3.12 Implications:**
- ⚠️ NVIDIA JetPack may not support Python 3.12 yet
- ⚠️ TensorFlow version tied to JetPack version
- ⚠️ May need to stay on Python 3.11 for Jetson
- ⚠️ PyTorch support depends on NVIDIA-provided wheels

**Recommendation:** Jetson platform may require remaining on Python 3.11 until JetPack updates

#### PC/macOS
**Current State:**
- `tensorflow==2.15.*`
- `tensorflow-metal` for macOS GPU acceleration
- Full ML stack supported

**Python 3.12 Implications:**
- ✅ TensorFlow 2.16+ works with Python 3.12
- ✅ `tensorflow-metal` compatible with TF 2.16+
- ✅ PyTorch 2.4+ fully supported
- ✅ No major blockers for development platforms

### Dependency Compatibility

**Core Dependencies (all Python 3.12 compatible):**
- ✅ numpy, pillow, docopt, tornado, requests
- ✅ PrettyTable, paho-mqtt, simple_pid
- ✅ pandas, pyyaml, psutil, pyserial

**Visualization Dependencies:**
- ✅ matplotlib - Python 3.12 compatible
- ⚠️ kivy - Check version for Python 3.12
- ✅ plotly - Python 3.12 compatible

**Hardware Dependencies:**
- ⚠️ Adafruit libraries - Check for Python 3.12 wheels
- ✅ gpiozero - Python 3.12 compatible
- ⚠️ picamera2 - Verify Python 3.12 support

---

## Code Changes Required for Python 3.12

### 1. Deprecation Issues

#### Keras Optimizer Parameters (HIGH PRIORITY)
**Location:** `donkeycar/parts/keras.py` lines 84-88

```python
# CURRENT (deprecated in TF 2.11+)
optimizer = keras.optimizers.Adam(lr=rate, decay=decay)
optimizer = keras.optimizers.SGD(lr=rate, decay=decay)
optimizer = keras.optimizers.RMSprop(lr=rate, decay=decay)

# REQUIRED CHANGE
optimizer = keras.optimizers.Adam(learning_rate=rate, weight_decay=decay)
optimizer = keras.optimizers.SGD(learning_rate=rate, weight_decay=decay)
optimizer = keras.optimizers.RMSprop(learning_rate=rate, weight_decay=decay)
```

**Impact:** Breaking change when upgrading to TensorFlow 2.16+

#### PyTorch Model Loading (MEDIUM PRIORITY)
**Location:** `donkeycar/parts/pytorch/ResNet18.py` line 14

```python
# CURRENT (deprecated in torchvision 0.13+)
model = models.resnet18(pretrained=True)

# REQUIRED CHANGE
from torchvision.models import ResNet18_Weights
model = models.resnet18(weights=ResNet18_Weights.IMAGENET1K_V1)
# OR for default weights:
model = models.resnet18(weights=ResNet18_Weights.DEFAULT)
```

**Impact:** Breaking change when upgrading PyTorch

### 2. Type Hint Updates

**Location:** `donkeycar/parts/interpreter.py` line 80

```python
# CURRENT (works but uses older syntax)
self.input_keys: list[str] = None

# RECOMMENDED (no change needed, already Python 3.9+ syntax)
# Keep as-is for Python 3.12
```

**Status:** ✅ Already compatible

### 3. TensorFlow Python API Changes

**Concern:** Uses `tensorflow.python.*` internal APIs
**Example:** `donkeycar/pipeline/training.py` line 7
```python
from tensorflow.python.keras.models import load_model
```

**Risk:** Internal APIs may change between TensorFlow versions

**Recommendation:** Migrate to public APIs:
```python
from tensorflow.keras.models import load_model
```

---

## PyTorch Migration Analysis

### Current State

**Existing PyTorch Infrastructure:**
- ✅ PyTorch training pipeline (`torch_train.py`)
- ✅ PyTorch data loading (`torch_data.py`)
- ✅ ResNet18 model implementation
- ✅ PyTorch Lightning integration
- ✅ Test coverage (`test_torch.py`)

**Integration Level:**
- Separate from main Keras pipeline
- Optional install (`[torch]` extra)
- Not integrated into main templates
- No deployment path (TFLite/TensorRT equivalent)

### Migration Complexity Assessment

#### Level 1: Model Architecture (MEDIUM COMPLEXITY)
**Keras Models to Convert:** 20+ model types
- Linear, Categorical, Localizer models
- RNN, LSTM, 3D CNN variants
- Behavioral models
- IMU-integrated models

**Effort:** Each model requires manual translation
- Map layer types (Conv2D → nn.Conv2d)
- Adjust parameter names and conventions
- Reimplement custom layers
- Validate numerical equivalence

**Estimated Effort:** 2-4 weeks per model type

#### Level 2: Training Pipeline (HIGH COMPLEXITY)
**Components to Migrate:**
1. Data loading and augmentation
   - Current: TensorFlow `tf.data.Dataset` API
   - Target: PyTorch `DataLoader` (partially done)
2. Training loop
   - Current: Keras `model.fit()` with callbacks
   - Target: PyTorch Lightning `Trainer` (partially done)
3. Callbacks and metrics
   - Early stopping, model checkpointing, TensorBoard
   - Most have Lightning equivalents
4. Mixed precision training
   - TensorFlow AMP → PyTorch AMP
5. Performance ranking (lap/segment-based)
   - Agnostic to framework, should work

**Estimated Effort:** 3-4 weeks

#### Level 3: Inference and Deployment (VERY HIGH COMPLEXITY)
**Major Challenges:**

1. **TFLite Replacement**
   - Current: Models converted to `.tflite` for edge devices
   - Options:
     - TorchScript (`.pt` files)
     - ONNX (cross-framework, but conversion complexity)
     - TensorFlow Lite converter from ONNX
   - **Issue:** No direct PyTorch → TFLite path
   - **Impact:** Raspberry Pi deployment affected

2. **TensorRT Replacement**
   - Current: TensorFlow SavedModel → TensorRT
   - PyTorch equivalent: TorchScript → TensorRT (via torch2trt)
   - **Issue:** Different deployment workflow
   - **Impact:** NVIDIA GPU inference

3. **Interpreter Architecture**
   - Current: `KerasInterpreter`, `TfLite`, `TensorRT` classes
   - Required: Refactor to support PyTorch, TorchScript
   - **Effort:** Significant architectural changes

4. **Model Format Changes**
   - Current: `.h5`, `.savedmodel`, `.tflite`
   - PyTorch: `.pt`, `.pth`, `.ckpt` (Lightning)
   - **Impact:** User migration path needed

**Estimated Effort:** 6-8 weeks

#### Level 4: Ecosystem Integration (MEDIUM COMPLEXITY)
**Components Affected:**
1. Template files (12+ templates)
   - Update model loading paths
   - Change default model types
2. CLI tools
   - `donkey train` command
   - `donkey makemovie` (uses TF for video)
3. Documentation
   - Installation guides
   - Training tutorials
   - Model selection guides

**Estimated Effort:** 2-3 weeks

### Total Effort Estimate
**Full PyTorch Migration:** 13-19 weeks (3-5 months)

### Dual Framework Approach (RECOMMENDED)

Instead of full migration, maintain both:

**Benefits:**
- ✅ Users can choose framework
- ✅ Gradual migration path
- ✅ Leverage existing TensorFlow ecosystem
- ✅ Experiment with PyTorch advantages

**Implementation:**
1. Improve PyTorch infrastructure
2. Add more model types to PyTorch
3. Create PyTorch deployment path
4. Update templates to support both
5. Maintain feature parity

**Estimated Effort:** 6-8 weeks initial, then incremental

---

## Migration Strategies

### Strategy 1: Python 3.12 Only (RECOMMENDED)

**Scope:** Update to Python 3.12 while keeping TensorFlow

**Steps:**
1. Update `setup.cfg` Python requirement to `>=3.11.0,<3.13`
2. Upgrade TensorFlow to 2.16+ (or 2.17)
3. Fix deprecated Keras optimizer parameters
4. Update CI/CD to test both Python 3.11 and 3.12
5. Test on all platforms (PC, macOS, Raspberry Pi)
6. Update documentation

**Timeline:** 1-2 weeks

**Risks:**
- ⚠️ TensorFlow 2.16 GPU detection issues
- ⚠️ Keras 3 breaking changes
- ⚠️ Raspberry Pi `tensorflow-aarch64` availability
- ⚠️ Jetson compatibility

**Mitigation:**
- Pin to TensorFlow 2.16.x (avoid 2.17 if unstable)
- Set Keras 2 compatibility mode if needed
- Thoroughly test GPU detection on all platforms
- Keep Python 3.11 support for Jetson

### Strategy 2: Python 3.12 + Enhanced PyTorch (MODERATE)

**Scope:** Python 3.12 + improve PyTorch support

**Steps:**
1. Execute Strategy 1 (Python 3.12 + TensorFlow 2.16)
2. Upgrade PyTorch to 2.4+
3. Fix deprecated `pretrained=` parameter
4. Add more model types to PyTorch
5. Create TorchScript/ONNX deployment path
6. Update templates to support both frameworks
7. Add framework selection to CLI

**Timeline:** 6-8 weeks

**Risks:**
- ⚠️ Dual framework maintenance burden
- ⚠️ User confusion about which to use
- ⚠️ Different deployment workflows

**Benefits:**
- ✅ Framework choice for users
- ✅ Gradual migration path
- ✅ Leverages PyTorch advantages where beneficial

### Strategy 3: Full PyTorch Migration (NOT RECOMMENDED)

**Scope:** Complete replacement of TensorFlow with PyTorch

**Reasons Against:**
1. **Time Investment:** 3-5 months of focused development
2. **Breaking Changes:** All existing models incompatible
3. **User Impact:** Forces migration for all users
4. **Edge Deployment:** TFLite replacement unclear
5. **Ecosystem Loss:** TensorFlow tooling well-established
6. **Risk:** High chance of bugs and regressions

**Only Consider If:**
- TensorFlow support for platform dropped
- Critical PyTorch-only features needed
- Full team buy-in for migration
- 6+ month timeline acceptable

---

## Platform-Specific Recommendations

### Raspberry Pi

**Recommended Approach:**
1. Test Python 3.12 with `tensorflow-aarch64==2.16.*`
2. Verify picamera2 compatibility
3. Test PyTorch 2.4+ as optional framework
4. Ensure 64-bit Raspberry Pi OS

**Fallback:**
- Keep Python 3.11 if TensorFlow 2.16 issues arise
- Use PyTorch as primary if TensorFlow unavailable

### Jetson Nano

**Recommended Approach:**
1. **Keep Python 3.11** until JetPack updates
2. Use NVIDIA-provided TensorFlow
3. Add PyTorch support from NVIDIA wheels
4. Test separately from main Python 3.12 migration

**Note:** Jetson may lag behind PC/RPi updates

### PC/macOS

**Recommended Approach:**
1. Lead with Python 3.12 migration
2. Upgrade to TensorFlow 2.16+
3. Upgrade PyTorch to 2.4+
4. Use as development/testing platform
5. Validate before deploying to edge devices

---

## Risks and Mitigations

### Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| TF 2.16 GPU issues | High | Medium | Pin to stable 2.16.x, extensive testing |
| Keras 3 breaks code | High | Low | Use Keras 2 compat mode, test thoroughly |
| RPi TF unavailable | High | Low | Prepare PyTorch fallback |
| Jetson incompatible | Medium | Medium | Keep Python 3.11 option for Jetson |
| Dependency conflicts | Medium | Medium | Use virtual environments, test matrix |
| User model migration | High | High (if full PyTorch) | Provide conversion tools, dual support |
| Performance regression | Medium | Low | Benchmark before/after |

### Community Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| User confusion | Medium | Clear documentation, migration guide |
| Support burden | Medium | Comprehensive testing, good error messages |
| Ecosystem fragmentation | High | Maintain backward compatibility |
| Documentation lag | Medium | Update docs in parallel with code |

---

## Dependency Update Requirements

### Minimum Changes (Python 3.12 Only)

**setup.cfg:**
```ini
[metadata]
classifiers =
    Programming Language :: Python :: 3.11
    Programming Language :: Python :: 3.12  # ADD

[options]
python_requires = >=3.11.0,<3.13  # CHANGE
install_requires =
    # ... existing packages unchanged

[options.extras_require]
pi =
    # ... existing packages
    tensorflow-aarch64==2.16.*  # CHANGE from 2.15.*

pc =
    tensorflow==2.16.*  # CHANGE from 2.15.*
    # ... rest unchanged

macos =
    tensorflow==2.16.*  # CHANGE from 2.15.*
    # ... rest unchanged

torch =
    torch==2.4.*  # CHANGE from 2.1.*
    pytorch-lightning>=2.5.0  # ADD version spec
    # ... rest unchanged
```

**CI/CD (.github/workflows/python-package-conda.yml):**
```yaml
- name: Create python 3.12 conda env  # CHANGE
  uses: conda-incubator/setup-miniconda@v3
  with:
    python-version: 3.12  # CHANGE
```

Add matrix testing:
```yaml
strategy:
  matrix:
    os: [macos-latest, ubuntu-latest]
    python-version: [3.11, 3.12]  # ADD
```

---

## Testing Strategy

### Phase 1: Python 3.12 Compatibility Testing
1. **Unit Tests:** Run full test suite on Python 3.12
2. **Integration Tests:** Test training pipeline end-to-end
3. **Platform Tests:** 
   - macOS (x86_64, ARM64)
   - Ubuntu 20.04, 22.04
   - Raspberry Pi OS (64-bit)
4. **GPU Tests:** Verify CUDA/Metal acceleration
5. **Model Tests:** Load and run existing models

### Phase 2: Framework Testing
1. **TensorFlow 2.16 Tests:** All model types
2. **PyTorch 2.4 Tests:** Existing PyTorch models
3. **Deployment Tests:** TFLite conversion
4. **Performance Tests:** Training speed, inference latency

### Phase 3: Regression Testing
1. **Existing Models:** Load and verify outputs
2. **Training Pipelines:** Compare metrics
3. **Hardware Integration:** Camera, IMU, actuators
4. **Web Interface:** Controller, telemetry

---

## Recommendations

### Immediate Actions (Priority 1)

1. **✅ APPROVE Python 3.12 Migration with TensorFlow 2.16**
   - Low risk, high compatibility
   - Keeps current architecture
   - Opens Python 3.12 features
   - Timeline: 1-2 weeks

2. **✅ Upgrade PyTorch to 2.4+ as Optional**
   - Improves optional PyTorch support
   - Python 3.12 compatible
   - No impact on TensorFlow users
   - Timeline: 1 week

3. **✅ Fix Deprecated API Usage**
   - Keras optimizer parameters
   - PyTorch `pretrained=` parameter
   - Prepares for future updates
   - Timeline: 1 week

### Medium-term Actions (Priority 2)

4. **Consider Dual Framework Support**
   - Enhance PyTorch infrastructure
   - Add more PyTorch model types
   - Create PyTorch deployment path
   - Maintain TensorFlow as default
   - Timeline: 6-8 weeks

5. **Platform-Specific Testing**
   - Raspberry Pi 4/5 with Python 3.12
   - Jetson compatibility assessment
   - Performance benchmarking
   - Timeline: 2-3 weeks

### Long-term Considerations (Priority 3)

6. **Evaluate Full PyTorch Migration**
   - Only if TensorFlow support degrades
   - Only with 6+ month timeline
   - Only with full team commitment
   - Requires user migration plan

7. **ONNX Integration**
   - Cross-framework model format
   - TensorFlow ↔ PyTorch conversion
   - Broader deployment options
   - Timeline: 4-6 weeks

---

## Conclusion

### Python 3.12 Migration: ✅ RECOMMENDED

**Verdict:** **PROCEED** with Python 3.12 migration

**Approach:**
- Upgrade to Python `>=3.11.0,<3.13`
- Upgrade TensorFlow to 2.16.x
- Upgrade PyTorch to 2.4.x (in `[torch]` extra)
- Fix deprecated API calls
- Extensive platform testing

**Risk Level:** **LOW-MEDIUM**

**Timeline:** 2-3 weeks for full implementation and testing

### PyTorch as Primary Framework: ⚠️ NOT RECOMMENDED (Now)

**Verdict:** **DEFER** full PyTorch migration

**Reasoning:**
1. Extensive TensorFlow integration (2,400+ lines)
2. Established edge deployment (TFLite, TensorRT)
3. High migration complexity (3-5 months)
4. Risk of breaking existing user workflows
5. PyTorch advantages don't outweigh migration cost

**Alternative:** **Dual Framework Support**
- Enhance existing PyTorch option
- Provide choice to users
- Gradual migration path
- Lower risk, incremental value

**Risk Level:** **HIGH** (for full migration)

### Final Recommendation

**Adopt Strategy 1 immediately:**
1. Update to Python 3.12
2. Upgrade TensorFlow to 2.16
3. Upgrade PyTorch to 2.4
4. Fix deprecation warnings
5. Comprehensive testing

**Consider Strategy 2 in next phase:**
1. Improve PyTorch support incrementally
2. Maintain TensorFlow as default
3. Offer framework choice
4. Evaluate user feedback

**Avoid Strategy 3 unless:**
1. TensorFlow support dropped
2. Critical PyTorch-only features needed
3. 6+ months available for migration

---

## Appendix: Version Compatibility Matrix

### TensorFlow Versions
| Version | Python 3.11 | Python 3.12 | Keras Version | Notes |
|---------|-------------|-------------|---------------|-------|
| 2.15.x | ✅ | ❌ | Keras 2 | Current donkeycar version |
| 2.16.x | ✅ | ✅ | Keras 3 (default) | First Python 3.12 support |
| 2.17.x | ✅ | ✅ | Keras 3 | Latest stable (2025) |

### PyTorch Versions
| Version | Python 3.11 | Python 3.12 | Notes |
|---------|-------------|-------------|-------|
| 2.1.x | ✅ | ❌ | Current donkeycar version |
| 2.2.x | ✅ | ✅ | First Python 3.12 support |
| 2.3.x | ✅ | ✅ | Improved compatibility |
| 2.4.x | ✅ | ✅ | Recommended for Python 3.12 |

### Platform Support
| Platform | Python 3.12 | TF 2.16 | PyTorch 2.4 | Notes |
|----------|-------------|---------|-------------|-------|
| PC (Linux/Win) | ✅ | ✅ | ✅ | Full support |
| macOS (Intel) | ✅ | ✅ | ✅ | Full support |
| macOS (ARM) | ✅ | ✅ | ✅ | Requires tensorflow-metal |
| Raspberry Pi 4/5 | ✅ | ✅* | ✅ | *Requires 64-bit OS |
| Jetson Nano | ⚠️ | ⚠️ | ⚠️ | Depends on JetPack version |

---

**Document Version:** 1.0  
**Author:** GitHub Copilot  
**Review Status:** Ready for technical review  
**Next Review:** After initial testing phase

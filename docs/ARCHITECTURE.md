# Eye-Response: System Architecture

**Version**: 1.0  
**Last Updated**: 2026-05-16

---

## 🏗️ System Architecture Overview

Eye-Response is composed of **8 independent but interconnected modules**, each responsible for a specific aspect of hands-free eye-tracking control.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Eye-Response Architecture                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │          Platform Layer (Android/iOS/Flutter)           │  │
│  └──────────────────────────────────────────────────────────┘  │
│           ↓                                      ↓               │
│  ┌─────────────────────┐         ┌──────────────────────────┐  │
│  │  UI Framework (4)   │         │ Safety & Ethics (8)      │  │
│  │ - Gaze Cursor       │         │ - Consent Layer          │  │
│  │ - Magnetic Button   │         │ - Privacy Controls       │  │
│  │ - Radial Menus      │         │ - Fallback Systems       │  │
│  │ - Gaze Keyboard     │         │ - Distraction Prevention │  │
│  └─────────────────────┘         └──────────────────────────┘  │
│           ↑                              ↑                       │
│  ┌─────────────────────┐         ┌──────────────────────────┐  │
│  │ Interaction Layer   │         │ Advanced Features (5)    │  │
│  │ (3)                 │         │ - Reading Mode           │  │
│  │ - Dwell Clicking    │         │ - Fatigue Monitor        │  │
│  │ - Smooth Pursuit    │         │ - Gaze Shortcuts         │  │
│  │ - Hover Detection   │         │ - Hands-Free Media       │  │
│  │ - Off-screen Rec.   │         │ - Notifications          │  │
│  └─────────────────────┘         └──────────────────────────┘  │
│           ↑                              ↑                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │    Blink Control (2) & Calibration (6)                   │  │
│  │ - Blink Detection    - Calibration Wizard                │  │
│  │ - Command Mapping    - User Profiles                     │  │
│  │ - Blink Rejection    - Drift Correction                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│           ↑                                      ↑                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │          Vision Engine (1) - Core Eye-Tracking          │   │
│  │ - Face Detection (MediaPipe Face Mesh)                  │   │
│  │ - Eye Pupil Tracking                                    │   │
│  │ - Gaze-to-Screen Mapping                                │   │
│  │ - Parallax Compensation                                 │   │
│  │ - Lighting Compensation                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│           ↑                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │   Native Device APIs (Camera, Accelerometer, etc.)      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📦 Module Details

### **Module 1: Vision Engine (Core Eye-Tracking)**

**Responsibility**: Real-time on-device computer vision pipeline

#### Components
```
Vision Engine
├── Face Detection
│   ├── MediaPipe Face Mesh (468 landmarks)
│   ├── Face orientation (pitch, yaw, roll)
│   └── Head movement tracking
├── Eye Tracking
│   ├── Iris center detection
│   ├── Pupil size monitoring
│   └── Gaze direction estimation
├── Screen Mapping
│   ├── Camera-to-screen transformation matrix
│   ├── Parallax compensation (edge regions)
│   ├── Perspective correction
│   └── Calibration point mapping
├── Image Processing
│   ├── Lighting normalization (auto-exposure)
│   ├── Contrast enhancement
│   ├── Frame stabilization (de-noising)
│   └── FPS management (30-60 FPS target)
└── Calibration Integration
    ├── 5-point calibration data
    ├── User-specific model fitting
    └── Drift adjustment
```

#### Key Algorithms

**Gaze Mapping**:
```
Screen_Position = Homography_Transform(Eye_Landmarks, Calibration_Points)
Parallax_Correction = Edge_Compensation_Factor * Distance_from_Center
Final_Gaze = Screen_Position + Parallax_Correction
```

**Head Movement Stabilization**:
```
Stable_Gaze = Smooth_Filter(Current_Gaze, Previous_Gaze, Head_Rotation)
```

#### Performance Targets
- **Accuracy**: 3-5° (relative to screen)
- **FPS**: 30-60 (adjustable for low-end devices)
- **Latency**: <150ms (camera capture → screen update)
- **CPU Usage**: <15% on mid-range devices

---

### **Module 2: Blink Control System**

**Responsibility**: Recognize blinks and map them to user commands

#### Components
```
Blink Control
├── Blink Detection
│   ├── Eye aspect ratio (EAR) calculation
│   ├── Blink onset/offset detection
│   ├── Blink duration measurement
│   └── Micro-blink filtering
├── Blink Classification
│   ├── Single Blink (150-400ms)
│   ├── Double Blink (within 800ms)
│   ├── Long Hold Blink (>600ms)
│   ├── Triple Blink (rapid succession)
│   └── Wink Detection (L eye vs. R eye)
├── Blink Rejection AI
│   ├── Natural blink baseline (15-20/min)
│   ├── Statistical outlier detection
│   ├── Machine learning classifier
│   └── False positive filtering
└── Command Mapping
    ├── Blink → Action interpreter
    ├── Command queue
    └── Feedback (audio/haptic)
```

#### Blink Type Specifications

| Type | Duration | Interval | Action | Notes |
|------|----------|----------|--------|-------|
| Single | 150-400ms | N/A | Click/Tap | Primary interaction |
| Double | 150-400ms each | <800ms | Right-click/Context | Secondary action |
| Long Hold | >600ms | N/A | Drag-and-drop start | Release = drop |
| Triple | 150-400ms each | <1000ms | Pause/Resume | Emergency toggle |
| Left Wink | 150-400ms | N/A | Back/Left nav | Single eye close |
| Right Wink | 150-400ms | N/A | Home/Right nav | Single eye close |

#### Blink Rejection Algorithm

```
Natural_Blink_Rate = 15-20 per minute
Current_Blink_Rate = Count(Blinks) / Time_Window

If Current_Blink_Rate > Threshold:
    Z_Score = (Current_Rate - Mean) / StdDev
    If Z_Score > 2.5:
        Mark as potential intentional blink
        Apply ML Classifier(Blink_Duration, EAR_Profile, Context)
```

#### Performance Targets
- **Detection Accuracy**: 95%+
- **False Positive Rate**: <5% (natural blinks)
- **Command Recognition**: 90%+ accuracy
- **Response Time**: <200ms (blink onset → action)

---

### **Module 3: Interaction Layer**

**Responsibility**: Handle dwell, gestures, and smooth tracking interactions

#### Components
```
Interaction Layer
├── Dwell Clicking
│   ├── Dwell time threshold (800ms)
│   ├── Progress ring visualization
│   ├── Target stability check
│   └── Dwell-to-click conversion
├── Smooth Pursuit Scrolling
│   ├── Floating UI element tracking
│   ├── Scroll velocity calculation
│   ├── Direction detection (up/down/left/right)
│   └── Momentum scrolling
├── Hover Detection
│   ├── Gaze-to-UI distance calculation
│   ├── Hover threshold (200ms)
│   ├── Hover state triggering
│   └── Fitts's Law optimization
├── Off-Screen Recovery
│   ├── Screen boundary detection
│   ├── Directional arrow display
│   ├── Auto-center on re-entry
│   └── Edge dead zones
└── Gesture Recognition
    ├── Saccade detection (rapid eye movement)
    ├── Eye gesture patterns (e.g., L→R = screenshot)
    └── Custom gesture mapping
```

#### Dwell Clicking Algorithm

```
For each frame:
    If Gaze_Position == Current_Target:
        Dwell_Time += Frame_Delta
        Draw Progress_Ring(Dwell_Time / 800ms)
        
        If Dwell_Time >= 800ms:
            Execute_Click(Current_Target)
            Reset_Dwell()
    Else:
        Reset_Dwell()
```

#### Smooth Pursuit Scrolling

```
Scroll_Velocity = Gaze_Position - UI_Element_Position
Scroll_Speed = Magnitude(Scroll_Velocity) * Sensitivity_Factor
Scroll_Direction = Normalize(Scroll_Velocity)

Apply_Scroll(Scroll_Direction, Scroll_Speed)
```

#### Performance Targets
- **Dwell Response**: 800ms ±50ms
- **Hover Detection**: 200ms ±30ms
- **Scroll Smoothness**: 60 FPS
- **Gesture Recognition**: 90%+ accuracy

---

### **Module 4: Gaze-Aware UI Framework**

**Responsibility**: Build accessible, gaze-native UI components

#### Components
```
UI Framework
├── Gaze Cursor
│   ├── Cursor rendering (smoothed position)
│   ├── Cursor customization (size, color, opacity)
│   ├── Parallax adjustment
│   └── Visible/hidden states
├── Magnetic Button
│   ├── Snap radius (40px default)
│   ├── Nearest element detection
│   ├── Snap animation
│   └── Haptic feedback
├── Radial Context Menu
│   ├── Menu generation from context
│   ├── Item arrangement (equidistant)
│   ├── Item highlighting
│   └── Selection via dwell/blink
├── Gaze-Optimized Keyboard
│   ├── Large key landing zones
│   ├── Look-to-type input
│   ├── Blink-to-confirm
│   ├── Word prediction (gaze-aware)
│   ├── Multi-language support
│   └── Accessibility profiles (large keys for tremor)
├── Dead Zone Manager
│   ├── System UI dead zones (notification bar, edges)
│   ├── Accidental activation prevention
│   ├── Dwell requirement for edge activation
│   └── Dynamic dead zone sizing
└── Hover Effects
    ├── UI element enlargement (Fitts's Law)
    ├── Color highlight changes
    ├── Shadow/depth effects
    └── Smooth state transitions
```

#### Magnetic Button Logic

```
For each interactive element:
    Distance = Distance_To_Gaze(Element_Bounds, Gaze_Position)
    
    If Distance <= Snap_Radius (40px):
        Highlight_Element()
        If Click_Action_Triggered:
            Execute_Element_Action()
```

#### Performance Targets
- **UI Responsiveness**: <100ms between gaze change and UI update
- **Cursor Smoothness**: 60 FPS
- **Keyboard Accuracy**: 95%+ character entry
- **Menu Rendering**: <50ms

---

### **Module 5: Advanced Features**

**Responsibility**: Premium productivity and accessibility features

#### Components
```
Advanced Features
├── Reading Mode
│   ├── Saccade pattern detection
│   ├── Reading speed estimation
│   ├── Auto-scroll speed calculation
│   ├── Pause on gaze-away
│   └── Reading statistics (optional)
├── Eye Fatigue Monitor
│   ├── Blink rate tracking
│   ├── Squinting detection
│   ├── Eye strain estimation
│   ├── 20-20-20 reminders
│   ├── Blue light reduction triggers
│   └── Break notifications
├── Quick Glance Notifications
│   ├── Notification preview (compact)
│   ├── Expand on 500ms dwell
│   ├── Blink-to-dismiss
│   ├── Priority filtering
│   └── Do-not-disturb integration
├── Gaze Shortcuts
│   ├── Eye gesture library (L→R, circles, etc.)
│   ├── Gesture recognition ML model
│   ├── Custom gesture mapping
│   ├── Gesture replay/undo
│   └── Shortcut dashboard
├── Hands-Free Media
│   ├── Photo capture (blink trigger)
│   ├── Focus point selection (gaze)
│   ├── Video recording (gaze start/stop)
│   ├── Gallery navigation (eye gestures)
│   └── Photo editing (gaze controls)
└── Accessibility Profiles (linked to Module 6)
    ├── Tremor Mode (heavy smoothing)
    ├── Low-Vision Mode (high contrast)
    └── One-Eye Operation
```

#### Reading Mode Algorithm

```
For each frame during reading mode:
    Detect_Saccades(Gaze_Data)  // Rapid eye movements
    
    If Reading_Pattern_Detected:
        Estimated_Reading_Speed = Analyze_Saccade_Patterns()
        Auto_Scroll_Velocity = Reading_Speed * Sensitivity
        Apply_Scroll(Auto_Scroll_Velocity)
        
        If Gaze_Leaves_Text_Area:
            Pause_Scrolling()
        Else:
            Resume_Scrolling()
```

#### Performance Targets
- **Reading Mode**: Smooth auto-scroll, <100ms latency
- **Fatigue Detection**: 95%+ accuracy
- **Gesture Recognition**: 90%+ accuracy
- **Notification Expansion**: <150ms

---

### **Module 6: Calibration & Personalization**

**Responsibility**: Accurate user-specific gaze mapping with minimal setup

#### Components
```
Calibration
├── Calibration Wizard
│   ├── 5-point calibration (corners + center)
│   ├── 30-second UX flow
│   ├── Real-time accuracy feedback
│   ├── Re-calibration option
│   └── Completion confirmation
├── User Profile
│   ├── Biometric data (eye geometry, device distance)
│   ├── Calibration matrix (per user)
│   ├── Device/phone orientation preferences
│   ├── Accessability profile settings
│   └── Encrypted local storage
├── Drift Correction
│   ├── Periodic accuracy checks
│   ├── Automated drift detection
│   ├── 2-hour reminder interval
│   ├── Quick 3-point micro-calibration
│   └── Drift trend analysis
├── Auto-Recalibration
│   ├── Missed click detection
│   ├── Accuracy threshold monitoring
│   ├── Automatic trigger (when accuracy drops)
│   ├── Minimal user interruption
│   └── Background recalibration
└── User Profile Management
    ├── Profile storage & retrieval
    ├── Profile deletion (privacy)
    ├── Profile sharing (optional, encrypted)
    └── Biometric template updates
```

#### 5-Point Calibration Flow

```
1. Display point at TOP-LEFT corner
   → User looks at it for 1 second
   → Record gaze sample (5 frames)
   
2. Display point at TOP-RIGHT corner
   → User looks for 1 second
   → Record gaze sample
   
3. Display point at CENTER
   → User looks for 1 second
   → Record gaze sample
   
4. Display point at BOTTOM-LEFT corner
   → User looks for 1 second
   → Record gaze sample
   
5. Display point at BOTTOM-RIGHT corner
   → User looks for 1 second
   → Record gaze sample

Compute_Calibration_Matrix(5 × Samples)
Validate_Accuracy(Test Clicks)
Save_User_Profile()
```

#### Drift Correction Algorithm

```
Every 2 hours:
    Display_Micro_Calibration_Prompt()
    
    If User_Accepts:
        Quick_3_Point_Calibration()
        Compare_With_Previous_Matrix()
        
        If Drift > Threshold:
            Apply_Correction_Matrix()
            Update_User_Profile()
        Else:
            No correction needed
```

#### Performance Targets
- **Calibration Time**: ~30 seconds
- **Accuracy Post-Calibration**: 3-5° (first-time user)
- **Profile Creation**: <500ms
- **Drift Detection**: 95%+ accuracy

---

### **Module 7: Platform Support**

**Responsibility**: Native platform implementations (Android, iOS)

#### Android (Kotlin)

```
Android Platform Layer
├── Native Modules
│   ├── Camera Access (CameraX)
│   ├── Face Landmark Detection (MediaPipe)
│   ├── GPU Delegate (TensorFlow Lite)
│   ├── Accelerometer Access (Sensor Framework)
│   └── Performance Monitoring
├── Architecture
│   ├── MainActivity (main UI thread)
│   ├── EyeResponseService (background service)
│   ├── CameraFrameProcessor (non-blocking)
│   ├── GazeUpdateListener (high-frequency callback)
│   └── SettingsActivity (preferences)
├── Permissions
│   ├── android.permission.CAMERA
│   ├── android.permission.ACCESS_FINE_LOCATION (accelerometer)
│   ├── android.permission.RECORD_AUDIO (optional)
│   └── android.permission.VIBRATE (haptic feedback)
└── Integration
    ├── Foreground service indicator
    ├── Doze mode optimization
    ├── Thermals monitoring
    └── Low-power mode support
```

#### iOS (Swift)

```
iOS Platform Layer
├── Native Modules
│   ├── AVCapture (camera)
│   ├── Vision Framework (face landmarks)
│   ├── CoreML (on-device inference)
│   ├── ARKit (optional)
│   ├── CoreMotion (accelerometer)
│   └── Performance Profiler
├── Architecture
│   ├── AppDelegate (lifecycle)
│   ├── ContentViewController (main UI)
│   ├── CameraSession (frame capture)
│   ├── GazeProcessor (non-blocking)
│   ├── SettingsViewController
│   └── AccessibilityProfilesViewController
├── Entitlements
│   ├── NSCameraUsageDescription
│   ├── NSMotionUsageDescription
│   └── Privacy manifest entries
└── Integration
    ├── Background activity monitoring
    ├── Thermal state tracking
    ├── Battery saving mechanisms
    └── iOS accessibility API integration
```

#### Performance Targets
- **Android**: <15% CPU on Snapdragon 600-series
- **iOS**: <15% CPU on A-series chips
- **Memory**: <200MB RAM during operation
- **Battery**: <10% drain per hour with continuous tracking

---

### **Module 8: Safety, Ethics & Privacy**

**Responsibility**: User consent, data privacy, and system safety

#### Components
```
Safety & Ethics Layer
├── Consent Indicator
│   ├── Green border glow (tracking active)
│   ├── Status bar icon
│   ├── Quick settings toggle
│   └── On-device storage notification
├── Privacy Controls
│   ├── Gaze data storage verification (none)
│   ├── Camera feed lifecycle audit
│   ├── Analytics opt-in/opt-out
│   ├── Data retention policy
│   └── GDPR/CCPA compliance
├── Fallback Systems
│   ├── Head Tracking (if face not detected)
│   ├── Voice Command Integration
│   ├── Manual Touch Input (always available)
│   └── Graceful degradation
├── Distraction Prevention
│   ├── Accelerometer-based walking detection
│   ├── Auto-pause during walking
│   ├── Ambient light detection (very low = likely sleeping)
│   └── Notification suppression
└── Audit & Logging
    ├── Interaction success rate metrics
    ├── Feature usage analytics (no personal data)
    ├── Performance monitoring
    ├── Error tracking
    └── User-triggered privacy reports
```

#### Consent Flow

```
On First Launch:
    Show Privacy Policy (simple, honest language)
    → Highlight "No gaze data leaves your device"
    → Show "What Eye-Response tracks" (only metrics)
    → Show "What Eye-Response does NOT track" (exact gaze points)
    
    User_Choice = Agree / Disagree
    
    If Agree:
        Enable Green Border Glow
        Start Eye-Response Service
        Store Consent Timestamp
    Else:
        Show Manual Input / Voice Command Options
```

#### Privacy Verification

```
On Every Session:
    1. Verify camera feed is NOT written to disk
    2. Verify gaze landmarks are processed only in RAM
    3. Verify ML inference is on-device (no network calls with gaze data)
    4. Verify analytics only track success/failure, not gaze positions
    5. Generate Privacy Report (on-demand)
```

#### Performance Targets
- **Consent Display**: <500ms
- **Fallback Activation**: <2 seconds
- **Privacy Audit**: <5 seconds

---

## 🔄 Data Flow

```
Camera Frame
    ↓
[Vision Engine]
    ├→ Face Detection (MediaPipe Face Mesh)
    ├→ Eye Pupil Tracking
    ├→ Gaze-to-Screen Mapping
    └→ Parallax Compensation
    ↓
Gaze Position (X, Y, Confidence)
    ↓
    ├→ [Blink Control]
    │   ├→ Blink Detection
    │   ├→ Blink Classification
    │   └→ Command Mapping
    │   ↓
    │   Blink Command
    │
    └→ [Interaction Layer]
        ├→ Dwell Clicking
        ├→ Smooth Pursuit
        ├→ Hover Detection
        └→ Off-Screen Recovery
        ↓
        Action (Click, Scroll, etc.)
        ↓
    ┌→ [UI Framework]
    │   ├→ Update Cursor
    │   ├→ Apply Magnetic Snapping
    │   ├→ Update UI State
    │   └→ Provide Feedback
    │
    └→ [Advanced Features]
        ├→ Reading Mode
        ├→ Fatigue Monitor
        ├→ Gesture Recognition
        └→ Notification Handler
        ↓
    [User Interaction Complete]
    ↓
    [Analytics] (success/failure metrics only)
```

---

## ⚡ Performance Optimization Strategy

### CPU Usage Optimization
- **MediaPipe GPU Delegate**: Use GPU for face mesh inference
- **Frame Skipping**: Process every Nth frame on low-end devices
- **Thread Management**: Off-load vision processing to background thread
- **Memory Management**: Recycle frame buffers, avoid allocation in hot path

### Battery Optimization
- **Adaptive FPS**: Drop to 15 FPS if device is in low-power mode
- **Auto-Suspend**: Stop processing when face not detected for 10 seconds
- **Sensor Fusion**: Use accelerometer to detect if phone is idle
- **Scheduled Tasks**: Defer non-critical tasks (analytics) to low-power times

### Latency Optimization
- **Pipelined Processing**: Process previous frame while capturing next
- **GPU Inference**: Use mobile GPU for ML inference
- **Early Termination**: Skip processing if face confidence too low
- **Interrupt Handling**: Prioritize gaze updates over other tasks

---

## 🧪 Testing Strategy

### Unit Tests
- Vision engine gaze mapping
- Blink detection algorithms
- Interaction layer calculations
- Calibration matrix computation

### Integration Tests
- Vision engine → Blink control integration
- Blink control → UI framework integration
- All modules end-to-end

### Performance Tests
- FPS benchmarks on various devices
- CPU/memory profiling
- Battery drain over 8 hours
- Thermal throttling response

### Accessibility Tests
- Tremor mode with simulated hand tremors
- Low-vision mode with accessibility features
- One-eye operation with single eye only
- Real user testing with motor disabilities

---

## 🔐 Security Considerations

1. **On-Device Processing**: No ML inference happens in the cloud
2. **No Gaze Storage**: Gaze positions are never written to disk
3. **Encrypted Profiles**: User calibration profiles encrypted locally
4. **Permission Isolation**: Camera permission strictly for vision processing
5. **No Telemetry**: No personal data sent to servers; only aggregated metrics

---

## 📚 Architecture Decision Records (ADRs)

### ADR-1: MediaPipe for Face Mesh
**Decision**: Use Google MediaPipe Face Mesh for 468-point face landmark detection  
**Rationale**: Pre-trained, optimized for mobile, open-source, on-device inference  
**Alternative**: Custom CNN model (rejected: more maintenance, less accuracy)

### ADR-2: On-Device Inference Only
**Decision**: All ML inference (face detection, eye tracking) happens on-device  
**Rationale**: Privacy, low latency, works offline, no network dependency  
**Alternative**: Cloud inference (rejected: violates privacy principle)

### ADR-3: Native Platform Implementation
**Decision**: Native Android (Kotlin) + Native iOS (Swift) instead of cross-platform  
**Rationale**: Optimal performance, platform APIs access, ecosystem best practices  
**Alternative**: Flutter (rejected: slightly higher latency, less control over native APIs)

---

## 🚀 Deployment Architecture

```
Development → Beta → Staging → Production

Development:
    └→ Local machine with Android emulator / iOS simulator

Beta:
    ├→ Internal TestFlight (iOS) / Play Store internal testing (Android)
    ├→ 50-100 real users
    └→ Feedback loop

Staging:
    ├→ Production-like environment
    ├→ Edge cases & stress testing
    └→ Performance benchmarks

Production:
    ├→ Apple App Store
    ├→ Google Play Store
    ├→ Staged rollout (5% → 25% → 100%)
    └→ Monitoring & telemetry
```

---

## 📞 Support & Documentation

- **User Guides**: Setup, calibration, accessibility profiles
- **Developer Docs**: API reference, integration guides, architecture deep-dives
- **API Documentation**: Generated from code comments (JSDoc/KDoc)
- **Architecture Docs**: This file, ADRs, design diagrams

---

**For detailed information on each module, see the respective documentation files in `docs/`.**

# StandAlone Scan - Quick Reference

## What Was Added?

### 1. New Button on Home Screen
- Between "Enterprise Scan" and "Security Advisor"
- Title: "STANDALONE SCAN"
- Subtitle: "ML-Powered Local Malware Detection"

### 2. Three New Screens

#### Screen 1: StandAlone Scan List
- Shows all installed apps (user or system)
- Each app shows:
  - App icon
  - App name
  - Package name
  - Size (MB)
  - **ML Prediction Score** (e.g., 45.2%)
  - **Threat Level Badge** (SAFE/SUSPICIOUS/MALICIOUS)
- Toggle buttons: "USER APPS" / "SYSTEM APPS"
- Refresh button to re-scan

#### Screen 2: StandAlone Detail
- Full app information (icon, name, package, size)
- **ML Prediction Section** with:
  - Threat score percentage
  - Color-coded threat label
- **Dangerous Permissions** list (same as Enterprise Scan)
- Action buttons: "App Settings" and "Uninstall"

#### Screen 3: ML Model Integration (Backend)
- Loads model from `assets/malware_detector_quantized.tflite`
- Loads features from `assets/selected_features.json`
- Extracts app features and runs predictions

---

## File Structure

```
NEW FILES:
✅ app/src/main/assets/selected_features.json
✅ app/src/main/java/.../util/MalwareDetector.kt
✅ app/src/main/java/.../ui/screens/StandAloneScanScreen.kt
✅ app/src/main/java/.../ui/screens/StandAloneDetailScreen.kt

MODIFIED FILES:
✅ app/src/main/java/.../model/AppInfo.kt
✅ app/src/main/java/.../ui/screens/HomeScreen.kt
✅ app/src/main/java/.../ui/screens/Navigation.kt
```

---

## How Predictions Work

1. **Feature Extraction**
   - Get all permissions app requests
   - Create binary vector (220 permissions + 1000 intents)
   - 1 if permission exists, 0 if not

2. **ML Prediction**
   - Feed feature vector to TFLite model
   - Model outputs threat score (0-1)
   - Apply sigmoid normalization

3. **Threat Classification**
   ```
   Score 0-33%   → SAFE (Green) ✓
   Score 33-67%  → SUSPICIOUS (Yellow) ⚠
   Score 67-100% → MALICIOUS (Red) ✗
   ```

---

## Color Coding

| Threat Level | Color | Score Range |
|---|---|---|
| SAFE | Green (#4CAF50) | 0-33% |
| SUSPICIOUS | Yellow (#FFC107) | 33-67% |
| MALICIOUS | Red (#FF5252) | 67-100% |
| UNKNOWN | Gray | Model error |

---

## App Categorization (Same as Enterprise Scan)

### User Apps
- Apps not marked as system (`ApplicationInfo.FLAG_SYSTEM`)
- Apps installed by user

### System Apps
- Official Android system apps (android.*, com.android.*)
- Official Google system apps (com.google.android.*)
- Filtered to exclude unreliable OEM packages

---

## Dangerous Permissions

Extracted using `PermissionInfo.PROTECTION_DANGEROUS` flag:
- Location
- Camera
- Microphone
- Contacts
- Calendar
- SMS/Phone
- Files
- etc.

---

## Key Points

✅ **Frontend Only**: All ML runs on device, no cloud calls
✅ **No Backend Changes**: Existing APIs unchanged
✅ **Consistent UI**: Matches Enterprise Scan design
✅ **Same Logic**: System app filtering identical to Enterprise Scan
✅ **Background Processing**: ML runs without blocking UI
✅ **User Friendly**: Clear visual indicators for threat levels

---

## User Journey

1. User taps "STANDALONE SCAN" on home
2. App list loads automatically
3. ML model runs on each app (background)
4. Each app shows prediction with color badge
5. User toggles between User/System apps
6. User taps any app to see detailed threat assessment
7. Can uninstall or open app settings from detail screen

---

## Status in Your Project

| Feature | Status |
|---|---|
| Home Screen Button | ✅ Done |
| StandAlone Scan Screen | ✅ Done |
| Detail Screen | ✅ Done |
| ML Model Integration | ✅ Done |
| Feature Extraction | ✅ Done |
| Navigation Routes | ✅ Done |
| JSON Features | ✅ Done |
| System App Logic | ✅ Done |
| Dangerous Permissions | ✅ Done |
| No Breaking Changes | ✅ Done |

---

## What's NOT Changed

- ❌ Enterprise Scan (fully functional)
- ❌ Security Advisor (fully functional)
- ❌ Backend APIs
- ❌ Database structure
- ❌ User interface style/theme
- ❌ AppDetailScreen for Enterprise Scan

---

## Ready to Use!

The StandAlone Scan feature is complete and ready for deployment! 🎉

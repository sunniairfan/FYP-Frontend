# 🚀 Quick Reference: Auto-Upload & Blocking Services

## Service Initialization

The services start automatically when the app launches in `MainActivity`:

```kotlin
// Services start here with device ID
val deviceId = Settings.Secure.getString(contentResolver, Settings.Secure.ANDROID_ID)
AutoUploadService.startAutoUploadListener(this)     // Polls every 5 seconds
BlockService.startBlockListener(this, deviceId)     // Polls every 3 seconds
```

---

## AutoUploadService Flow

### Backend Sends Request
```json
GET /api/block-upload/auto-upload-requests
Response: {
  "requests": [{
    "id": "auto_upload_com.evil.app_123",
    "appName": "Evil App",
    "packageName": "com.evil.app",
    "detectedEngines": 45,
    "status": "pending"
  }]
}
```

### Service Processes Request
```
1️⃣ Update Status: "in_progress"
   PUT /api/block-upload/auto-upload-requests/auto_upload_com.evil.app_123
   Body: { "status": "in_progress" }

2️⃣ Get APK File from Device
   /data/app/com.evil.app-xyz/base.apk

3️⃣ Upload to Backend
   POST /uploadapp
   Files: [APK Binary]
   Metadata: { appName, packageName, source: "auto_upload_high_risk" }

4️⃣ Mark Completed
   PUT /api/block-upload/auto-upload-requests/auto_upload_com.evil.app_123
   Body: {
     "status": "completed",
     "apkFileName": "com.evil.app_1234567890.apk"
   }
```

### Duplicate Prevention
✅ Same request ID won't be uploaded twice  
✅ Uses `completedRequests` Set to track  
✅ Automatic cleanup on service restart

---

## BlockService Flow

### Backend Sends Request
```json
GET /api/block-upload/block-requests/device/{deviceId}
Response: {
  "requests": [{
    "id": "block_com.evil.app_456",
    "appName": "Evil App",
    "packageName": "com.evil.app",
    "detectedEngines": 62,
    "status": "pending",
    "detailsForUser": {
      "threat_level": "CRITICAL",
      "why_blocked": "Detected by 62 antivirus engines"
    }
  }]
}
```

### Service Processes Request
```
1️⃣ Update Status: "processing"
   PUT /api/block-upload/block-requests/block_com.evil.app_456
   Body: { "status": "processing" }

2️⃣ Disable App on Device
   PackageManager.setApplicationEnabledSetting(
     "com.evil.app",
     COMPONENT_ENABLED_STATE_DISABLED_USER
   )

3️⃣ Show Alert to User
   🛑 CRITICAL SECURITY ALERT
   Evil App has been BLOCKED
   Threat Level: CRITICAL
   Detected by: 62 antivirus engines
   [OK Button]

4️⃣ Mark Completed
   PUT /api/block-upload/block-requests/block_com.evil.app_456
   Body: {
     "status": "completed",
     "blocked_at": "2024-01-02T10:35:30Z",
     "deviceId": "device-id-123"
   }
```

### Double-Block Prevention
✅ Same package won't be blocked twice  
✅ Uses `blockedApps` Set to track  
✅ Already-blocked apps are skipped

---

## Configuration

### Polling Intervals
- **AutoUploadService**: Every 5 seconds
- **BlockService**: Every 3 seconds

### Backend Endpoints
```
Base URL: http://192.168.137.1:5000

Auto-Upload:
  GET    /api/block-upload/auto-upload-requests
  PUT    /api/block-upload/auto-upload-requests/{requestId}
  POST   /uploadapp

Blocking:
  GET    /api/block-upload/block-requests/device/{deviceId}
  PUT    /api/block-upload/block-requests/{requestId}
```

---

## Monitoring Logs

### Filter by service
```bash
# Auto-Upload Service logs
adb logcat | grep "AutoUploadService"

# Block Service logs
adb logcat | grep "BlockService"

# Both services
adb logcat | grep -E "AutoUploadService|BlockService"
```

### Key Log Messages
```
✅ AUTO-UPLOAD COMPLETED: [AppName]
✅ APP BLOCKED: [AppName]
❌ Upload failed for [PackageName]
⏭️ Skipping already processed
⏭️ Skipping already blocked
```

---

## Manual Control (Optional)

### Stop Services
```kotlin
AutoUploadService.stopAutoUploadListener()
BlockService.stopBlockListener()
```

### Unblock App (if needed)
```kotlin
BlockService.unblockApp("com.example.app")
```

### Get Blocked Apps List
```kotlin
val blockedApps = BlockService.getBlockedApps()
blockedApps.forEach { packageName ->
    Log.i("BlockedApps", packageName)
}
```

---

## File Locations

| Component | Path |
|-----------|------|
| AutoUploadService | `util/AutoUploadService.kt` |
| BlockService | `util/BlockService.kt` |
| MainActivity (Integration) | `MainActivity.kt` |
| This Guide | `IMPLEMENTATION_GUIDE.md` |

---

## Test Commands

### Trigger Auto-Upload (from backend)
```bash
curl -X GET http://device-ip:5000/api/block-upload/auto-upload-requests
```

### Trigger App Blocking (from backend)
```bash
curl -X GET http://device-ip:5000/api/block-upload/block-requests/device/{deviceId}
```

### Check Device ID
```bash
adb shell settings get secure android_id
```

---

## Status Codes

### Upload Request Status
- `pending` → Waiting to be processed
- `in_progress` → Service is uploading
- `completed` → APK uploaded successfully
- `failed` → Upload encountered error

### Block Request Status
- `pending` → Waiting to be processed
- `processing` → Service is blocking app
- `completed` → App successfully blocked
- `failed` → Block encountered error

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Apps not uploading | Check network connection, verify backend URL |
| Apps not blocking | Check device ID matches, verify permissions |
| Duplicate uploads | Clear app data and restart |
| Service not starting | Check MainActivity logs, verify context |
| Alerts not showing | Check system alert permissions, verify UI thread |

---

**Last Updated**: April 23, 2026  
**Version**: 1.0 Production

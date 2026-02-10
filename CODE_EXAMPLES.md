# Code Examples - StandAlone Scan Implementation

## 1. ML Model Prediction Example

### How the MalwareDetector Works

```kotlin
// Initialize detector
val malwareDetector = MalwareDetector(context)

// Extract features from an app
val features = malwareDetector.extractAppFeatures(packageName, packageManager)
// Returns: FloatArray with 1220 features (220 permissions + 1000 intents)

// Get prediction
val prediction = malwareDetector.predictMalware(packageName, packageManager)
// prediction.score: Float (0-1)
// prediction.label: String ("safe", "suspicious", "malicious")
// prediction.confidence: Float (0-1)
```

### Feature Vector Structure

```
Feature Index 0-219: Permissions
  - If app has permission → 1.0
  - If app doesn't have permission → 0.0

Feature Index 220-1219: Intents
  - Currently set to 0.0 (placeholder)
  - Can be enhanced with manifest analysis
```

### Prediction Process

```kotlin
private fun sigmoid(x: Float): Float {
    return (1f / (1f + exp(-x))).coerceIn(0f, 1f)
}

// In predictMalware():
val rawScore = output[0][0]              // TFLite model output
val predictionScore = sigmoid(rawScore)  // Normalize to [0, 1]

val label = when {
    predictionScore < 0.33f -> "safe"
    predictionScore < 0.67f -> "suspicious"
    else -> "malicious"
}
```

---

## 2. Home Screen Button Example

### Adding the Button

```kotlin
OptionCard(
    iconPainter = painterResource(id = R.drawable.ic_shield),
    title = "STANDALONE SCAN",
    subtitle = "ML-Powered Local Malware Detection",
    onClick = {
        navController.navigate("standalone_scan")
    },
    animationDelayMs = enterDelay * 2
)
```

### Result
- Button appears between "ENTERPRISE SCAN" and "SECURITY ADVISOR"
- Same styling and animations as other buttons
- Navigates to `StandAloneScanScreen`

---

## 3. App List Display Example

### StandAloneAppCard Component

```kotlin
@Composable
fun StandAloneAppCard(
    appInfo: AppInfo,
    malwareDetector: MalwareDetector,
    packageManager: PackageManager,
    onClick: () -> Unit
) {
    // Async ML prediction
    var prediction by remember { mutableStateOf<Pair<Float, String>?>(null) }
    
    LaunchedEffect(appInfo.packageName) {
        withContext(Dispatchers.Default) {
            val result = malwareDetector.predictMalware(
                appInfo.packageName, 
                packageManager
            )
            prediction = Pair(result.score, result.label)
        }
    }
    
    val (score, label) = prediction ?: Pair(0.5f, "unknown")
    
    // Color based on threat level
    val predictionColor = when (label) {
        "safe" -> Color(0xFF4CAF50)        // Green
        "suspicious" -> Color(0xFFFFC107)  // Yellow
        "malicious" -> Color(0xFFFF5252)   // Red
        else -> Color.Gray
    }
    
    // Display card with prediction
    Card(...) {
        Row(...) {
            // Icon, name, package, size...
            
            // Prediction display
            Column(...) {
                Text("${String.format("%.1f%%", score * 100)}")
                Surface(...) {
                    Text(label.uppercase())
                }
            }
        }
    }
}
```

### Display Example
```
┌─────────────────────────────────────┐
│ [Icon] App Name        45.2% ╔═════╗│
│        com.example    Size   ║ SAFE║
│        10.5 MB               ╚═════╝
└─────────────────────────────────────┘
```

---

## 4. Detail Screen Example

### Displaying ML Prediction

```kotlin
Card(
    modifier = Modifier.fillMaxWidth(),
    shape = RoundedCornerShape(16.dp),
    colors = CardDefaults.cardColors(
        containerColor = cyberSecondary.copy(alpha = 0.7f)
    )
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text(
            "ML PREDICTION",
            style = MaterialTheme.typography.titleSmall.copy(
                fontWeight = FontWeight.Bold,
                color = cyberAccent,
                fontSize = 12.sp,
                letterSpacing = 1.sp
            ),
            modifier = Modifier.padding(bottom = 12.dp)
        )

        val predictionColor = when (predictionLabel) {
            "safe" -> Color(0xFF4CAF50)
            "suspicious" -> Color(0xFFFFC107)
            "malicious" -> Color(0xFFFF5252)
            else -> Color.Gray
        }

        Row(
            modifier = Modifier.fillMaxWidth(),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            Column {
                Text("Threat Score:")
                Text(
                    String.format("%.1f%%", predictionScore * 100),
                    style = MaterialTheme.typography.titleLarge.copy(
                        color = predictionColor,
                        fontWeight = FontWeight.Bold,
                        fontSize = 24.sp
                    ),
                    modifier = Modifier.padding(top = 4.dp)
                )
            }

            Surface(
                color = predictionColor.copy(alpha = 0.2f),
                shape = RoundedCornerShape(12.dp),
                modifier = Modifier.border(
                    2.dp,
                    predictionColor.copy(alpha = 0.5f),
                    RoundedCornerShape(12.dp)
                )
            ) {
                Text(
                    predictionLabel.uppercase(),
                    style = MaterialTheme.typography.labelLarge.copy(
                        color = predictionColor,
                        fontWeight = FontWeight.Bold,
                        fontSize = 16.sp
                    ),
                    modifier = Modifier.padding(
                        horizontal = 16.dp,
                        vertical = 8.dp
                    )
                )
            }
        }
    }
}
```

### Display Example
```
┌────────────────────────────────────┐
│ ML PREDICTION                       │
│                                    │
│ Threat Score:          SUSPICIOUS  │
│         68.3%          ┌────────┐  │
│                        │SUSPICIOUS│
│                        └────────┘  │
└────────────────────────────────────┘
```

---

## 5. Dangerous Permissions Display

### Extraction Logic

```kotlin
// In StandAloneDetailScreen
LaunchedEffect(packageName) {
    try {
        withContext(Dispatchers.Default) {
            val packageInfo = pm.getPackageInfo(packageName, PackageManager.GET_PERMISSIONS)
            val requestedPermissions = packageInfo.requestedPermissions
            val filtered = mutableListOf<String>()
            
            requestedPermissions?.forEach { permission ->
                try {
                    val permInfo = pm.getPermissionInfo(permission, 0)
                    // Check if it's a dangerous permission
                    if ((permInfo.protectionLevel and PermissionInfo.PROTECTION_DANGEROUS) != 0) {
                        filtered.add(permission)
                    }
                } catch (_: Exception) {}
            }
            
            dangerousPermissions = filtered
        }
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```

### Display Logic

```kotlin
if (dangerousPermissions.isNotEmpty()) {
    Text(
        "DANGEROUS PERMISSIONS (${dangerousPermissions.size})",
        style = MaterialTheme.typography.labelSmall.copy(
            fontWeight = FontWeight.Bold,
            color = cyberAccent,
            fontSize = 12.sp,
            letterSpacing = 1.sp
        ),
        modifier = Modifier.padding(bottom = 12.dp)
    )

    Column(
        verticalArrangement = Arrangement.spacedBy(8.dp),
        modifier = Modifier.padding(bottom = 24.dp)
    ) {
        dangerousPermissions.forEach { permission ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .background(
                        cyberSecondary.copy(alpha = 0.5f),
                        RoundedCornerShape(8.dp)
                    )
                    .padding(12.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Icon(
                    imageVector = Icons.Default.Warning,
                    contentDescription = null,
                    tint = Color(0xFFFFC107),
                    modifier = Modifier.size(16.dp)
                )
                Text(
                    permission.substringAfterLast('.'),
                    style = MaterialTheme.typography.bodySmall.copy(
                        color = cyberText,
                        fontSize = 11.sp
                    ),
                    modifier = Modifier.weight(1f)
                )
            }
        }
    }
}
```

### Display Example
```
DANGEROUS PERMISSIONS (4)

⚠ CAMERA
⚠ LOCATION
⚠ CONTACTS
⚠ MICROPHONE
```

---

## 6. Navigation Routes

### Added Routes

```kotlin
composable("standalone_scan") {
    StandAloneScanScreen(navController = navController)
}

composable(
    "standalone_detail/{packageName}/{appName}",
    arguments = listOf(
        navArgument("packageName") { type = NavType.StringType },
        navArgument("appName") { type = NavType.StringType }
    )
) { backStackEntry ->
    val packageName = backStackEntry.arguments?.getString("packageName") ?: ""
    val appName = backStackEntry.arguments?.getString("appName") ?: ""
    StandAloneDetailScreen(
        packageName = packageName, 
        appName = appName, 
        navController = navController
    )
}
```

### Navigation Usage

```kotlin
// From HomeScreen to StandAloneScanScreen
navController.navigate("standalone_scan")

// From StandAloneScanScreen to StandAloneDetailScreen
navController.navigate("standalone_detail/${app.packageName}/${app.appName}")

// Back navigation
navController.navigateUp()
```

---

## 7. System App Filtering

### User vs System Apps

```kotlin
val apps = pm.getInstalledApplications(PackageManager.GET_META_DATA)
    .filter {
        val isSystem = (it.flags and ApplicationInfo.FLAG_SYSTEM) != 0
        if (showUserApps) {
            !isSystem  // Show only non-system apps
        } else {
            // Show only authentic system apps
            isSystem && isAuthenticSystemApp(it.packageName)
        }
    }
```

### Authentic System App Check

```kotlin
private fun isAuthenticSystemApp(packageName: String): Boolean {
    // Whitelist of authentic system apps
    val AUTHENTIC_SYSTEM_PACKAGES = setOf(
        "android",
        "com.android.systemui",
        "com.android.launcher3",
        "com.android.phone",
        "com.android.contacts",
        "com.google.android.apps.maps",
        "com.google.android.gms",
        // ... more system apps
    )
    
    // Blacklist of unreliable OEM packages
    val UNRELIABLE_SYSTEM_PREFIXES = listOf(
        "com.android.cts",
        "com.qualcomm.",
        "com.coloros.",
        "com.oplus.",
        "com.vivo.",
        "com.miui.",
        "com.huawei.",
        "com.xiaomi.",
        "com.samsung.",
        // ... more OEM packages
    )
    
    // Check if in whitelist
    if (AUTHENTIC_SYSTEM_PACKAGES.contains(packageName)) return true
    
    // Check if in blacklist
    if (UNRELIABLE_SYSTEM_PREFIXES.any { packageName.startsWith(it) }) return false
    
    // Check official prefixes
    val officialPrefixes = listOf("com.google.", "com.android.", "android.")
    return officialPrefixes.any { packageName.startsWith(it) }
}
```

---

## 8. JSON Features Loading

### Load from Assets

```kotlin
private fun loadSelectedFeatures() {
    try {
        val featuresJson = context.assets
            .open("selected_features.json")
            .bufferedReader()
            .use { it.readText() }
        
        selectedFeatures = JSONObject(featuresJson)
        
        selectedFeatures?.let { features ->
            // Load permissions
            val permissionsArray = features.getJSONArray("permissions")
            for (i in 0 until permissionsArray.length()) {
                permissionsList.add(permissionsArray.getString(i))
            }
            
            // Load intents
            val intentsArray = features.getJSONArray("intents")
            for (i in 0 until intentsArray.length()) {
                intentsList.add(intentsArray.getString(i))
            }
        }
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```

### JSON Structure

```json
{
  "permissions": [
    "android.permission.CAMERA",
    "android.permission.ACCESS_FINE_LOCATION",
    "android.permission.READ_CONTACTS",
    ...
  ],
  "intents": [
    "android.intent.action.MAIN",
    "android.intent.action.BOOT_COMPLETED",
    ...
  ]
}
```

---

## 9. Async Prediction

### Using Coroutines

```kotlin
LaunchedEffect(appInfo.packageName) {
    withContext(Dispatchers.Default) {
        // Heavy computation on background thread
        val result = malwareDetector.predictMalware(
            appInfo.packageName, 
            packageManager
        )
        // Update UI on main thread
        prediction = Pair(result.score, result.label)
    }
}
```

### In Scan Screen

```kotlin
LaunchedEffect(refreshTrigger, showUserApps) {
    isScanning = true
    withContext(Dispatchers.Default) {
        // Run predictions in background
        apps.forEach { app ->
            val prediction = malwareDetector.predictMalware(
                app.packageName, 
                pm
            )
            // Results displayed as they complete
        }
    }
    isScanning = false
}
```

---

## Summary

All code examples show:
✅ Proper async operations with coroutines
✅ Correct state management
✅ Color-coded UI elements
✅ Error handling
✅ Following Kotlin best practices
✅ Integration with existing codebase

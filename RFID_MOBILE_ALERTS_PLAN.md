# RFID Mobile Alerts - Development Plan

**Important:** This document is an architecture and planning specification. No hardware, backend, or mobile application code is implemented yet. This plan describes what we will build.

---

## 1. Project Overview

This project adds mobile phone notifications to an existing RFID door-lock system. When someone scans an RFID card at the door, the system will send an instant notification to an Android phone. This lets you know who is entering your home or office in real-time.

The notification will tell you:
- Whether the card was authorized (allowed) or unauthorized (not allowed)
- The name of the person (if authorized)
- The exact time they tried to enter
- Whether the door actually unlocked or remained locked

This is useful for:
- Parents tracking when children come home
- Office managers monitoring access
- Security teams detecting unauthorized access attempts
- Homeowners getting alerts when someone arrives

---

## 2. Hardware Requirements

### What You Already Have:
- **Arduino Uno** - The main microcontroller that controls the door lock
- **EM-18 RFID Reader (125 kHz UART)** - Reads RFID cards and sends the card ID to the Arduino
- **16x2 LCD Display** - Shows messages like "Access Granted" or "Access Denied"
- **Servo Motor (SG90 or MG995)** - Physically moves the door lock mechanism to lock/unlock
- **Power Supply** - Powers the Arduino and connected components

### What You Need to Add:
- **Wi-Fi Module** - Either:
  - **ESP8266 NodeMCU** (cheaper, good for beginners)
  - **ESP32** (more powerful, built-in Wi-Fi and Bluetooth)
- **Logic Level Shifter** (bi-directional) OR resistors for voltage divider - To safely connect 5V Arduino to 3.3V ESP
- **Jumper wires and breadboard** - For connecting components
- **USB cable** - For programming the ESP module
- **5V power supply** - To power both Arduino and ESP module

---

## 3. Connectivity Approach: ESP32 vs. ESP8266

You have two options to add Wi-Fi to your existing system:

### Option A: Keep Arduino Uno + Add ESP8266/ESP32
- **How it works:** Arduino continues running your existing code. You add an ESP8266 or ESP32 connected to Arduino via serial (TX/RX pins). Arduino sends card scan data to ESP, and ESP sends it to the internet.
- **Pros:**
  - You don't have to rewrite your existing Arduino code
  - Familiar hardware (Arduino Uno stays the same)
  - Good if your current code is complex and working well
- **Cons:**
  - Two microcontrollers means more complexity
  - Need to handle serial communication between Arduino and ESP
  - Requires voltage level conversion (5V to 3.3V)
  - More wiring and potential points of failure

### Option B: Replace Arduino with Single ESP32
- **How it works:** Move all logic from Arduino to ESP32. The ESP32 reads the RFID reader, controls the LCD, moves the servo, and connects to Wi-Fi.
- **Pros:**
  - Only one microcontroller to program
  - ESP32 is more powerful than Arduino Uno (faster, more memory)
  - Built-in Wi-Fi means no serial communication needed
  - 3.3V logic matches ESP32, no voltage conversion needed
  - Better for long-term project growth
- **Cons:**
  - You need to rewrite your existing Arduino code for ESP32
  - Slightly different pin configuration (need to check datasheet)
  - ESP32 uses 3.3V (some components may need 5V, like some LCDs)

### **Recommendation for Beginners:**

**Go with Option B - Replace Arduino with ESP32.**

**Why:**
1. **Simpler:** One board to program instead of two
2. **No voltage conversion:** ESP32 is 3.3V, so you don't need a logic level shifter between microcontrollers
3. **More capable:** ESP32 has more memory and speed for future features
4. **Built-in Wi-Fi:** No need to manage serial communication between boards
5. **Code transfer is easy:** The C++ code for Arduino can be adapted to ESP32 with minimal changes

The learning curve for moving from Arduino to ESP32 is small, and you'll have a cleaner, more professional system.

---

## 4. Hardware Communication & Voltage Safety

### The Voltage Problem:
- **Arduino Uno** operates at **5V logic levels**
- **ESP8266 and ESP32** operate at **3.3V logic levels**

If you connect them directly:
- ESP's TX pin (3.3V) connected to Arduino's RX pin (5V) is **SAFE** - 3.3V is enough for Arduino to recognize as "high"
- Arduino's TX pin (5V) connected to ESP's RX pin (3.3V) is **DANGEROUS** - 5V can damage the ESP

### Safe Wiring Options:

#### Option 1: Bi-Directional Logic Level Shifter (Recommended)
A logic level shifter module automatically converts voltage between 5V and 3.3V in both directions.

**Wiring:**
- HV (High Voltage) side: Connect to Arduino (5V)
- LV (Low Voltage) side: Connect to ESP (3.3V)
- TX1 (Arduino TX) → HV1 → LV1 → RX (ESP RX)
- TX (ESP TX) → LV2 → HV2 → RX1 (Arduino RX)
- Connect HV to Arduino 5V pin
- Connect LV to ESP 3.3V pin
- Connect GND of both boards together

#### Option 2: Resistor Voltage Divider (Cheaper)
Use two resistors to drop 5V to 3.3V. Use this only if connecting Arduino TX to ESP RX.

**Formula:**
- Resistor 1: 1kΩ (from Arduino TX to ESP RX)
- Resistor 2: 2kΩ (from ESP RX to ground)
- This creates ~3.3V at the ESP RX pin

**Wiring:**
```
Arduino TX (5V) ---[1kΩ]--- ESP RX (3.3V)
                            |
                          [2kΩ]
                            |
                           GND
```

**Important:**
- You still need a direct connection from ESP TX to Arduino RX (3.3V is safe for Arduino)
- Always connect GND of both boards together
- Only needed if using Option A (Arduino + ESP module)
- If using Option B (ESP32 only), this is not needed

### Wiring for Option B (ESP32 Only):
Since ESP32 is 3.3V, check your components:
- **EM-18 RFID Reader:** Works with 3.3V or 5V - connect to ESP32 3.3V
- **16x2 LCD:** Most work with 5V - you may need a level shifter or I2C adapter
- **Servo Motor:** Works with 5V - power from external 5V supply, signal from ESP32

---

## 5. System Architecture

Here is the complete data flow from card scan to phone notification:

```
1. USER ACTION
   └─ User places RFID card near EM-18 reader

2. HARDWARE LAYER
   └─ EM-18 RFID Reader reads card UID (unique ID)
   └─ EM-18 sends UID via UART serial to ESP32
   └─ ESP32 receives UID and looks it up in authorized list
   └─ ESP32 decides: authorized or unauthorized?
   └─ ESP32 controls servo (unlock if authorized, stay locked if not)
   └─ ESP32 displays message on LCD ("Access Granted" or "Access Denied")

3. COMMUNICATION LAYER
   └─ ESP32 prepares data packet: {card_id, status, timestamp, device_id}
   └─ ESP32 connects to Wi-Fi (if not already connected)
   └─ ESP32 sends HTTPS POST request to Firebase Cloud Function

4. CLOUD LAYER (Firebase)
   └─ Cloud Function receives the request
   └─ Cloud Function validates the request (security check)
   └─ Cloud Function writes access log to Firestore Database
   └─ Cloud Function triggers FCM (Firebase Cloud Messaging) notification

5. NOTIFICATION LAYER
   └─ FCM sends push notification to all registered Android devices
   └─ Android app receives notification
   └─ Android app shows alert on screen:
       - High-priority warning for unauthorized access
       - Friendly notification for authorized access
   └─ Android app updates access history list in UI

6. DATABASE LAYER
   └─ Firestore stores access log document:
       {
         card_id: "A1B2C3D4",
         user_name: "John Doe",
         status: "authorized",
         timestamp: "2026-10-30T14:30:00Z",
         device_id: "rfid-door-01"
       }
   └─ Android app can fetch history from Firestore
```

### Key Points:
- The door lock works locally (even without internet)
- Cloud logging is a "nice to have" - not required for door to work
- Notifications are sent after the door decision is made
- All data is timestamped at the server for accuracy
- Multiple Android phones can receive notifications

---

## 6. Android App Architecture

### Technology Recommendation: Kotlin

**Why Kotlin instead of Java?**
- **Modern and safer:** Kotlin has built-in features that prevent common bugs (null pointer exceptions, etc.)
- **Less code:** Kotlin can do the same thing as Java with 30-50% less code
- **Official Google choice:** Google recommends Kotlin for new Android apps
- **Better tooling:** Android Studio has excellent Kotlin support
- **Future-proof:** Java is older, Kotlin is the future of Android

### Architecture Pattern: MVVM (Model-View-ViewModel)

**What is MVVM?**
- **Model:** Data layer (Firestore database, local cache)
- **View:** UI layer (Jetpack Compose screens)
- **ViewModel:** Logic layer (connects Model and View, handles business logic)

**Why MVVM?**
- Separates concerns - easier to test and maintain
- UI updates automatically when data changes
- Survives screen rotations (phone turning)
- Standard Android architecture

### UI Framework: Jetpack Compose

**What is Jetpack Compose?**
- Modern way to build Android UI using code (not XML)
- Like React for web, but for Android
- Less boilerplate code than old XML layouts

**Why Compose?**
- Faster development - write UI in Kotlin code
- Live preview in Android Studio
- Less code to maintain
- Google's recommended approach for new apps

### App Structure:
```
android-app/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/rfidalerts/
│   │   │   │   ├── MainActivity.kt
│   │   │   │   ├── model/ (Data classes)
│   │   │   │   ├── viewmodel/ (ViewModels)
│   │   │   │   ├── ui/ (Compose screens)
│   │   │   │   ├── service/ (Firebase messaging service)
│   │   │   │   └── utils/ (Helper functions)
│   │   │   ├── res/ (Resources)
│   │   │   └── AndroidManifest.xml
│   │   └── build.gradle.kts
│   └── google-services.json (Firebase config - add manually, never commit)
└── build.gradle.kts
```

### Key Android Features to Use:
- **Jetpack Compose** - For UI screens
- **ViewModel** - For screen logic
- **LiveData/StateFlow** - For reactive data updates
- **Firebase SDK** - For Firestore and FCM
- **Coroutines** - For asynchronous operations
- **Room Database** - Optional local cache for offline viewing

---

## 7. Backend and Database Recommendation

### Technology: Google Firebase (Serverless)

**What is Firebase?**
Firebase is a cloud platform by Google that provides ready-to-use backend services. You don't need to set up your own server.

**Why Firebase instead of a custom VPS server?**

| Aspect | Custom VPS Server | Firebase |
|--------|------------------|----------|
| **Cost** | $5-20/month for a basic server | Free tier (generous limits) |
| **Setup** | Install Linux, configure Nginx, secure server | Click "Create Project" - done in 5 minutes |
| **Maintenance** | Security updates, backups, scaling | Google handles everything |
| **Scalability** | Manual scaling needed | Automatic scaling |
| **Beginner-friendliness** | Complex | Very easy |
| **Database** | Install MySQL, configure | Built-in Firestore |
| **API** | Write your own REST API | Built-in SDKs |

### Firebase Services We'll Use:

#### 1. **Firestore Database**
- What: NoSQL cloud database
- Why: Real-time sync, offline support, easy to use
- Cost: Free tier includes 50K reads/day, 20K writes/day
- Use case: Store access logs, authorized user list

#### 2. **Cloud Functions**
- What: Server-side code that runs in response to events
- Why: Validate data, trigger notifications, business logic
- Cost: Free tier includes 125K invocations/month
- Use case: Receive ESP32 data, write to Firestore, trigger FCM

#### 3. **Firebase Cloud Messaging (FCM)**
- What: Push notification service
- Why: Reliable, free, works on Android
- Cost: Free
- Use case: Send notifications to Android app

#### 4. **Firebase Authentication** (Optional)
- What: User login system
- Why: If you want multiple people to log into the app
- Cost: Free
- Use case: Admin login to manage authorized cards

### Firestore Data Structure:

**Collection: access_logs**
```
Document ID: auto-generated
{
  card_id: "A1B2C3D4",
  user_name: "John Doe",
  status: "authorized", // or "unauthorized"
  timestamp: "2026-10-30T14:30:00Z",
  device_id: "rfid-door-01",
  notification_sent: true
}
```

**Collection: authorized_cards**
```
Document ID: card ID (e.g., "A1B2C3D4")
{
  user_name: "John Doe",
  added_at: "2026-10-01T10:00:00Z",
  active: true
}
```

### Cloud Function Logic:
```javascript
// When ESP32 sends data via HTTP POST
exports.logAccess = functions.https.onRequest(async (req, res) => {
  const { card_id, status, device_id } = req.body;

  // Validate request (security)
  if (!validateRequest(req)) {
    return res.status(403).send('Unauthorized');
  }

  // Write to Firestore
  await admin.firestore().collection('access_logs').add({
    card_id,
    user_name: await getUserName(card_id),
    status,
    timestamp: admin.firestore.FieldValue.serverTimestamp(),
    device_id
  });

  // Send FCM notification
  await sendNotification(card_id, status);

  res.status(200).send('Logged');
});
```

### Security Rules (Firestore):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Only the ESP32 can write access logs
    match /access_logs/{logId} {
      allow create: if request.auth != null;
      allow read: if request.auth != null;
    }

    // Only app admins can manage authorized cards
    match /authorized_cards/{cardId} {
      allow read, write: if request.auth != null;
    }
  }
}
```

---

## 8. Push Notification Design

### Firebase Cloud Messaging (FCM) Integration

FCM is Google's free push notification service. It works reliably on Android.

### Notification Types:

#### 1. **Authorized Access (Informational)**
- **Priority:** Normal
- **Sound:** Gentle notification sound
- **Content:**
  - Title: "Door Unlocked"
  - Body: "John Doe entered at 2:30 PM"
  - Icon: Green checkmark
  - Action: Tap to view access history

#### 2. **Unauthorized Access (High Priority)**
- **Priority:** High
- **Sound:** Loud, urgent alarm sound
- **Vibration:** Strong vibration pattern
- **Content:**
  - Title: "⚠️ UNAUTHORIZED ACCESS"
  - Body: "Unknown card tried to enter at 2:30 PM"
  - Icon: Red warning triangle
  - Action: Tap to view details
  - Additional: Optional SMS/email alert (future feature)

### Android App Notification Handling:

#### Foreground (App is open):
- Show an in-app banner at the top of the screen
- Update the access history list automatically
- Play sound if unauthorized access
- Vibrate phone if unauthorized access

#### Background (App is closed or minimized):
- Show system notification in notification tray
- Tap notification opens the app to access history
- Sound and vibration based on priority

#### Implementation Steps:
1. Add Firebase Messaging dependency to Android app
2. Create `FirebaseMessagingService` class
3. Override `onMessageReceived()` for foreground messages
4. Override `onNewToken()` to handle device token changes
5. Register device token with Firestore when app first opens
6. Handle notification channels (required for Android 8.0+)

### Code Example (Android - Kotlin):
```kotlin
class MyFirebaseMessagingService : FirebaseMessagingService() {
    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        val notification = remoteMessage.notification
        val data = remoteMessage.data

        if (notification != null) {
            // Show notification
            showNotification(
                notification.title ?: "RFID Alert",
                notification.body ?: "New access event",
                data["type"] ?: "info"
            )
        }
    }

    private fun showNotification(title: String, body: String, type: String) {
        val channelId = if (type == "unauthorized") "high_priority" else "normal"
        // Create and show notification...
    }
}
```

### FCM Message Format (from Cloud Function):
```javascript
const message = {
  notification: {
    title: status === 'authorized' ? 'Door Unlocked' : '⚠️ UNAUTHORIZED ACCESS',
    body: status === 'authorized'
      ? `${user_name} entered at ${time}`
      : `Unknown card tried to enter at ${time}`
  },
  data: {
    type: status,
    card_id: card_id,
    timestamp: timestamp
  },
  token: deviceToken // Android device's FCM token
};
```

---

## 9. Authorized vs. Unauthorized Access Handling

### Authorized Access Flow:

1. **Card Scan:** User scans authorized card
2. **Local Decision:** ESP32 checks local authorized list
3. **Door Action:** Servo moves to unlock position
4. **LCD Display:** Shows "Access Granted - John Doe"
5. **Cloud Logging:** Send to Firestore (if internet available)
6. **Notification:** Send FCM to Android phones
7. **Android App Shows:**
   - Green notification with friendly message
   - "Door Unlocked - John Doe entered at 2:30 PM"
   - Normal notification sound
   - Update to access history list

### Unauthorized Access Flow:

1. **Card Scan:** User scans unknown card
2. **Local Decision:** ESP32 checks local authorized list - not found
3. **Door Action:** Servo stays in locked position
4. **LCD Display:** Shows "Access Denied"
5. **Cloud Logging:** Send to Firestore (if internet available)
6. **Notification:** Send HIGH PRIORITY FCM to Android phones
7. **Android App Shows:**
   - Red warning notification
   - "⚠️ UNAUTHORIZED ACCESS - Unknown card at 2:30 PM"
   - Loud alarm sound
   - Strong vibration pattern
   - Update to access history list with red highlight
8. **Optional:** The app could also:
   - Send an SMS to the owner
   - Send an email alert
   - Play a siren sound if app is open

### Visual Feedback:

**Authorized:**
- Notification color: Green
- Icon: Checkmark or unlocked padlock
- Sound: Gentle chime
- Vibration: Single short pulse

**Unauthorized:**
- Notification color: Red
- Icon: Warning triangle or locked padlock
- Sound: Loud alarm or siren
- Vibration: Long, repeated pattern
- Action: Could enable "panic mode" in app

### Priority Levels:
- **Authorized:** Normal priority (won't wake phone if in Do Not Disturb)
- **Unauthorized:** High priority (bypasses Do Not Disturb in some cases)

---

## 10. Access History and Timestamps

### Data Schema for Access Logs

Every access event (authorized or unauthorized) will be stored in Firestore with this structure:

```javascript
{
  // Document ID: Auto-generated by Firestore
  card_id: "A1B2C3D4",           // The RFID card's unique ID
  user_name: "John Doe",         // Name if authorized, "Unknown" if not
  status: "authorized",          // Either "authorized" or "unauthorized"
  timestamp: "2026-10-30T14:30:00Z",  // Server timestamp (UTC)
  device_id: "rfid-door-01",     // Which door (if you have multiple)
  notification_sent: true,       // Whether FCM was sent
  retry_count: 0                 // If cloud send failed, retry attempts
}
```

### Timestamp Handling:

**Why Use Server Timestamps?**
- ESP32 clock may not be accurate (no real-time clock)
- Server timestamp ensures accurate time across all devices
- Timezone handling is automatic
- Prevents tampering (user can't change time on ESP32)

**Firestore Server Timestamp:**
```javascript
// In Cloud Function
timestamp: admin.firestore.FieldValue.serverTimestamp()
```

### Access History in Android App:

**Features:**
1. **List View:** Show all access events, newest first
2. **Filtering:** Filter by authorized only, unauthorized only, or all
3. **Search:** Search by user name or card ID
4. **Date Range:** View history for specific day/week/month
5. **Export:** Export to CSV (future feature)
6. **Offline Cache:** View recent history even without internet

**UI Design:**
```
┌─────────────────────────────────┐
│  Access History          [Filter]│
├─────────────────────────────────┤
│ Today, Oct 30                   │
│ ┌─────────────────────────────┐ │
│ │ 🟢 John Doe                 │ │
│ │ 2:30 PM - Authorized        │ │
│ │ Door: Front                 │ │
│ └─────────────────────────────┘ │
│ ┌─────────────────────────────┐ │
│ │ 🔴 Unknown Card             │ │
│ │ 1:15 PM - Unauthorized      │ │
│ │ Door: Front                 │ │
│ └─────────────────────────────┘ │
│ Yesterday, Oct 29              │
│ ┌─────────────────────────────┐ │
│ │ 🟢 Jane Smith               │ │
│ │ 8:00 AM - Authorized        │ │
│ │ Door: Front                 │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

### Data Retention:
- Keep access logs for 1 year (can be configured)
- Older logs can be archived or deleted via Cloud Function
- This keeps database size manageable on free tier

---

## 11. Offline and Network Disconnection Behavior

### The Golden Rule: **Door Must Always Work Locally**

Even if the internet is completely down, the door lock MUST still unlock for authorized cards. Security and safety come first.

### ESP32 Local Storage:

**Store Authorized Cards Locally:**
- Save the list of authorized card IDs in ESP32's flash memory (Preferences library)
- On startup, ESP32 loads this list
- Periodically (e.g., daily), ESP32 syncs with Firestore to update the list
- If sync fails, use the last known good list

**Example:**
```cpp
// ESP32 code
#include <Preferences.h>

Preferences preferences;

void setup() {
  preferences.begin("rfid", false);
  // Check if authorized list exists in flash
  if (!preferences.isKey("auth_list")) {
    // Load default list
  }
}

bool isAuthorized(String cardId) {
  // Check local list - no internet needed
  String authList = preferences.getString("auth_list", "");
  return authList.indexOf(cardId) >= 0;
}
```

### Queue System for Offline Events:

**When Internet is Down:**
1. ESP32 still accepts/denies cards locally
2. ESP32 stores access events in a local queue (max 100 events)
3. When Wi-Fi reconnects, ESP32 sends queued events to Firestore
4. After successful send, clear from queue

**Queue Structure:**
```cpp
struct AccessEvent {
  String card_id;
  String status;
  unsigned long timestamp; // ESP32's local time
};

AccessEvent eventQueue[100];
int queueIndex = 0;
```

**Reconnection Logic:**
```cpp
void loop() {
  // Check if Wi-Fi is connected
  if (WiFi.status() == WL_CONNECTED) {
    // Send queued events
    for (int i = 0; i < queueIndex; i++) {
      sendToFirestore(eventQueue[i]);
    }
    queueIndex = 0; // Clear queue after sending
  }

  // Normal card scanning logic...
}
```

### Android App Offline Behavior:

**When App is Offline:**
1. App shows "No internet connection" warning
2. App cannot fetch new access history from Firestore
3. App can still show cached recent history (last 50 events)
4. Notifications may be delayed until phone reconnects
5. App stores incoming notifications locally

**When Phone Reconnects:**
1. App automatically syncs with Firestore
2. Updates access history list
3. Shows any missed notifications

### Failure Modes and Fallbacks:

| Failure Scenario | What Happens | Fallback |
|------------------|--------------|----------|
| Wi-Fi router down | ESP32 cannot connect to internet | Door still works locally, events queued |
| Firebase down | Cloud function not responding | ESP32 retries, events queued |
| Android phone offline | No notifications | App shows last known history, syncs when online |
| ESP32 loses power | Queue lost | Use battery backup or UPS (optional) |
| Firestore full | Cannot write new logs | Implement automatic cleanup of old logs |

### User Experience:

**Authorized User During Outage:**
- Scans card → Door unlocks immediately
- LCD shows "Access Granted"
- No notification (internet down)
- When internet returns → Notification sent (optional: "John Doe entered earlier at 2:30 PM")

**Unauthorized User During Outage:**
- Scans card → Door stays locked
- LCD shows "Access Denied"
- No notification (internet down)
- When internet returns → Warning notification sent with timestamp

---

## 12. Development Roadmap

### Phase 1: Hardware Breadboarding and Serial Communication (Week 1-2)

**Goals:**
- Set up ESP32 and test basic functionality
- Connect EM-18 RFID reader to ESP32
- Test reading RFID cards
- Connect LCD display to ESP32
- Connect servo motor to ESP32
- Test complete door lock logic locally

**Tasks:**
1. Install ESP32 board support in Arduino IDE
2. Wire ESP32 to breadboard with power supply
3. Connect EM-18 RFID reader (VCC, GND, TX, RX)
4. Write code to read card IDs from EM-18
5. Test with multiple RFID cards
6. Connect 16x2 LCD (I2C recommended for fewer wires)
7. Write code to display messages on LCD
8. Connect servo motor (VCC, GND, Signal)
9. Write code to move servo (0° = locked, 90° = unlocked)
10. Combine everything: scan card → check list → move servo → show on LCD
11. Test authorized and unauthorized cards
12. Test rapid scanning (multiple cards quickly)

**Deliverable:**
- Working prototype on breadboard
- Arduino sketch for ESP32 that controls door lock locally
- Test report showing it works without internet

---

### Phase 2: Cloud Connection and Database Logging (Week 3-4)

**Goals:**
- Connect ESP32 to Wi-Fi
- Set up Firebase project
- Create Firestore database
- Write Cloud Function to receive data
- Send data from ESP32 to Firebase
- Verify data appears in Firestore

**Tasks:**
1. Create Firebase project at console.firebase.google.com
2. Enable Firestore database
3. Create collections: access_logs, authorized_cards
4. Write security rules for Firestore
5. Set up Cloud Functions (Node.js)
6. Write HTTP endpoint to receive ESP32 data
7. Add Wi-Fi code to ESP32 sketch
8. Test ESP32 Wi-Fi connection
9. Write ESP32 code to send HTTP POST to Cloud Function
10. Test end-to-end: scan card → check in Firestore
11. Add retry logic for failed Wi-Fi sends
12. Test offline behavior (disconnect Wi-Fi, scan card, reconnect)

**Deliverable:**
- Firebase project with Firestore and Cloud Functions
- ESP32 code that sends data to cloud
- Test report showing data logging works

---

### Phase 3: Android UI and FCM Push Notifications (Week 5-7)

**Goals:**
- Create Android app project
- Set up Firebase in Android app
- Implement FCM to receive notifications
- Build UI screens (list view, detail view)
- Connect to Firestore to fetch access history
- Test notifications from Cloud Function

**Tasks:**
1. Install Android Studio
2. Create new Android project (Kotlin, Jetpack Compose)
3. Add Firebase SDK to app
4. Add google-services.json (download from Firebase)
5. Create FirebaseMessagingService
6. Implement notification channels
7. Build main screen with access history list
8. Connect to Firestore to fetch access logs
9. Implement real-time updates (Firestore listener)
10. Add filtering (authorized/unauthorized)
11. Test FCM notifications from Cloud Function
12. Test foreground and background notifications
13. Add notification sound and vibration
14. Test unauthorized access high-priority alert

**Deliverable:**
- Working Android app
- Notifications working on Android phone
- Access history viewable in app

---

### Phase 4: Field Testing, Security Hardening, and Enclosure Assembly (Week 8)

**Goals:**
- Test the complete system in real environment
- Fix any bugs found during testing
- Add security hardening
- Build permanent enclosure
- Document everything

**Tasks:**
1. Install ESP32 and components in permanent location
2. Test with real door/lock mechanism
3. Test for several days with regular use
4. Fix any bugs discovered
5. Add security: validate API requests, encrypt data
6. Review and tighten Firestore security rules
7. Design and 3D print or build enclosure
8. Mount components in enclosure
9. Add proper strain relief for cables
10. Write user manual
11. Create setup guide for adding new cards
12. Document all code with comments
13. Final system test

**Deliverable:**
- Fully assembled and installed system
- User documentation
- Code documentation
- Final test report

---

### Optional Future Enhancements:

**Phase 5: Advanced Features (Future)**
- Add camera module to take photo on unauthorized access
- Add multiple door support
- Add admin web dashboard
- Add SMS/email alerts for unauthorized access
- Add temporary access codes (time-limited cards)
- Add biometric fingerprint reader as secondary auth
- Add voice control integration
- Add smart home integration (Alexa/Google Home)

---

## 13. Testing Strategy

### Unit Testing (Each Component Individually)

**Hardware Testing:**
1. **EM-18 RFID Reader:**
   - Test with 10+ different cards
   - Test rapid scanning (scan same card 5 times in 10 seconds)
   - Test card at different distances from reader
   - Test card at different angles

2. **LCD Display:**
   - Test displaying long messages
   - Test special characters
   - Test display speed

3. **Servo Motor:**
   - Test movement to locked position (0°)
   - Test movement to unlocked position (90°)
   - Test returning to center
   - Test 100+ cycles for reliability

4. **ESP32 Wi-Fi:**
   - Test connection to different Wi-Fi networks
   - Test reconnection after Wi-Fi router restart
   - Test with weak Wi-Fi signal
   - Test with Wi-Fi password change

**Software Testing:**
1. **Authorized Card Check:**
   - Test known authorized card → should unlock
   - Test unknown card → should deny
   - Test empty card ID → should deny
   - Test card ID with special characters → should handle

2. **Queue System:**
   - Test with 1 event in queue
   - Test with 100 events in queue (max)
   - Test queue overflow (101st event)
   - Test queue persistence after power cycle

### Integration Testing (All Components Together)

**End-to-End Flow:**
1. **Happy Path (Authorized Card):**
   - Scan authorized card
   - Verify: Servo unlocks
   - Verify: LCD shows "Access Granted"
   - Verify: Data sent to Firestore
   - Verify: Android notification received
   - Verify: Notification shows correct name and time

2. **Unauthorized Path:**
   - Scan unknown card
   - Verify: Servo stays locked
   - Verify: LCD shows "Access Denied"
   - Verify: Data sent to Firestore
   - Verify: Android high-priority notification received
   - Verify: Notification shows warning

3. **Offline Behavior:**
   - Disconnect Wi-Fi
   - Scan authorized card
   - Verify: Door still unlocks
   - Verify: Event queued
   - Reconnect Wi-Fi
   - Verify: Queued event sent to Firestore
   - Verify: Notification received

4. **Rapid Scanning:**
   - Scan 5 cards in 10 seconds
   - Verify: All events logged
   - Verify: All notifications received
   - Verify: No data loss

### Android App Testing:

**Notification Testing:**
1. Test notification when app is open (foreground)
2. Test notification when app is closed (background)
3. Test notification when phone is locked
4. Test notification sound for authorized vs unauthorized
5. Test vibration pattern for unauthorized
6. Test tapping notification opens app
7. Test notification when phone is in Do Not Disturb mode

**UI Testing:**
1. Test access history list loads correctly
2. Test pull-to-refresh
3. Test filter by authorized only
4. Test filter by unauthorized only
5. Test search by name
6. Test tapping item shows details
7. Test offline cache shows recent history
8. Test orientation change (portrait/landscape)

### Network Testing:

**Latency Testing:**
1. Measure time from card scan to notification
2. Test with good Wi-Fi signal
3. Test with weak Wi-Fi signal
4. Test with cellular data on phone
5. Target: Notification within 3 seconds in good conditions

**Failure Testing:**
1. Test with Firebase service down (simulate)
2. Test with invalid API key
3. Test with expired Cloud Function
4. Test with corrupted data from ESP32
5. Verify graceful error handling

### Security Testing:

1. Test with fake card ID
2. Test with malformed HTTP request to Cloud Function
3. Test with stolen API token
4. Test Firestore security rules
5. Test if ESP32 can write to unauthorized collections
6. Test if Android app can read/write without authentication

### Field Testing (Real Environment):

1. Install in actual door location
2. Test for 1 week with regular use
3. Monitor for false positives/negatives
4. Test with different users
5. Test at different times of day
6. Monitor Wi-Fi stability
7. Test battery/power backup if used
8. Gather user feedback

### Performance Testing:

1. Test system for 24 hours continuous operation
2. Monitor ESP32 temperature
3. Monitor memory usage
4. Test with 1000+ access logs in Firestore
5. Test Android app with large history list
6. Test Cloud Function response time under load

### Regression Testing:

After any code change, re-run:
- All hardware tests
- End-to-end flow tests
- Offline behavior tests
- Notification tests

---

## 14. Security Best Practices

### Firestore Security Rules

**Principle:** Least privilege - only allow necessary access.

**Access Logs Collection:**
```
match /access_logs/{logId} {
  // Only authenticated users can read
  allow read: if request.auth != null;

  // Only our Cloud Function can write (not directly from app)
  allow create: if request.auth.token.email == 'cloud-function@rfid-system.iam.gserviceaccount.com';
}
```

**Authorized Cards Collection:**
```
match /authorized_cards/{cardId} {
  // Only authenticated admin users can read/write
  allow read, write: if request.auth != null
    && request.auth.token.email == 'admin@yourdomain.com';
}
```

### API Key and Secret Protection

**Never Commit Secrets:**
- `.env` files - Add to `.gitignore`
- `google-services.json` - Add to `.gitignore`, add manually to project
- Firebase service account keys - Never commit
- Wi-Fi passwords - Store in ESP32 code (compiled), not in source files
- API tokens - Use environment variables

**Best Practices:**
```cpp
// DON'T DO THIS (commits password to Git)
const char* wifiPassword = "MyPassword123";

// DO THIS (use secure storage or separate config file)
// Store in separate untracked file:
// config.h (add to .gitignore)
const char* wifiPassword = WIFI_PASSWORD;
```

### ESP32 Security

**Validate Incoming Data:**
```cpp
bool isValidCardId(String cardId) {
  // Check length
  if (cardId.length() != 8) return false;

  // Check for only hex characters
  for (int i = 0; i < cardId.length(); i++) {
    char c = cardId.charAt(i);
    if (!isHexadecimalDigit(c)) return false;
  }

  return true;
}
```

**HTTPS Only:**
- Always use HTTPS for Cloud Function calls
- Never send data over plain HTTP
- Validate SSL certificates

**Physical Security:**
- Enclose ESP32 in locked box
- Hide UART lines inside enclosure
- Use tamper-evident seals
- Consider adding physical lock to prevent tampering

### Cloud Function Security

**Validate Request Source:**
```javascript
// Check for API key or specific headers
const apiKey = req.headers['x-api-key'];
if (apiKey !== process.env.ESP32_API_KEY) {
  return res.status(403).send('Forbidden');
}
```

**Rate Limiting:**
- Prevent abuse by limiting requests per minute
- Use Firebase's built-in rate limiting or implement custom

**Data Sanitization:**
```javascript
// Never trust incoming data
const cardId = sanitizeInput(req.body.card_id);
const status = ['authorized', 'unauthorized'].includes(req.body.status)
  ? req.body.status
  : 'unauthorized';
```

### Android App Security

**Secure Storage:**
- Use Android Keystore for sensitive data
- Use SharedPreferences for non-sensitive data
- Never store passwords in plain text

**Network Security:**
- Use Android's Network Security Configuration
- Enable certificate pinning for production
- Validate SSL certificates

**App Authentication:**
- Require Firebase Authentication to access app
- Implement proper logout
- Clear sensitive data on logout

### Wi-Fi Security

**Use WPA2 or WPA3:**
- Don't use open Wi-Fi networks
- Use strong Wi-Fi password
- Change default router password

**Separate IoT Network:**
- Create separate Wi-Fi network for IoT devices
- Isolate from main network using VLAN or guest network
- Prevent IoT devices from accessing other devices

### OTA Updates Security

If implementing Over-The-Air updates for ESP32:
- Sign firmware updates with cryptographic key
- Verify signature before applying update
- Only allow updates from trusted server

### Monitoring and Alerts

**Set Up Alerts:**
- Monitor Cloud Function error rate
- Alert on high unauthorized access attempts
- Monitor Firestore for unusual patterns
- Set up budget alerts for Firebase usage

### Regular Security Audits

- Review security rules quarterly
- Rotate API keys annually
- Update dependencies regularly
- Review Firebase project audit logs
- Test security after any changes

### User Security Best Practices

**For the End User:**
1. Keep Android app updated
2. Use strong password for phone
3. Enable phone lock (fingerprint, PIN, pattern)
4. Don't share access cards
5. Report lost cards immediately
6. Regularly review access history
7. Change Wi-Fi password periodically

---

## Appendix: Beginner-Friendly Setup Instructions

### Quick Start Guide

**Step 1: Gather Hardware**
- ESP32 development board
- EM-18 RFID reader
- 16x2 LCD with I2C adapter
- Servo motor (SG90 or MG995)
- Jumper wires
- Breadboard
- 5V power supply
- USB cable for ESP32

**Step 2: Install Software**
- Install Arduino IDE from arduino.cc
- In Arduino IDE: File → Preferences → Additional Boards Manager URLs
- Add: `https://dl.espressif.com/dl/package_esp32_index.json`
- Tools → Board → Boards Manager → Search "ESP32" → Install
- Select board: Tools → Board → ESP32 Arduino → ESP32 Dev Module

**Step 3: Wire Components**
Follow wiring diagram in Phase 1 documentation.

**Step 4: Test Hardware**
- Upload test sketch to ESP32
- Open Serial Monitor (115200 baud)
- Scan card - should see card ID printed

**Step 5: Set Up Firebase**
- Go to console.firebase.google.com
- Create new project
- Enable Firestore
- Add Android app
- Download google-services.json

**Step 6: Install Android Studio**
- Download from developer.android.com/studio
- Create new project (Empty Activity)
- Add Firebase SDK
- Copy google-services.json to app folder

**Step 7: Test End-to-End**
- Scan card
- Check door unlocks
- Check phone receives notification

### Troubleshooting Common Issues

**ESP32 won't connect to Wi-Fi:**
- Check Wi-Fi password is correct
- Check ESP32 is within range of router
- Try restarting ESP32
- Check router is 2.4GHz (ESP32 doesn't support 5GHz)

**LCD shows garbage:**
- Check I2C address (usually 0x27 or 0x3F)
- Check wiring (SDA, SCL, VCC, GND)
- Try I2C scanner sketch to find address

**Servo doesn't move:**
- Check external power supply (ESP32 can't power servo directly)
- Check signal wire is connected to correct pin
- Check servo isn't damaged

**No notification on phone:**
- Check phone has internet
- Check app has notification permissions enabled
- Check Firebase Cloud Function is deployed
- Check FCM token is registered in Firestore

**Door won't unlock for authorized card:**
- Check card ID is in authorized list
- Check ESP32 serial monitor for errors
- Check servo is wired correctly
- Check logic (isAuthorized function)

---

## Conclusion

This plan provides a complete roadmap for building a professional RFID door-lock system with mobile notifications. The system prioritizes:

1. **Reliability:** Door works even without internet
2. **Security:** Proper authentication and access control
3. **User Experience:** Clear notifications and history
4. **Beginner-Friendly:** Uses simple, well-documented technologies
5. **Scalability:** Serverless architecture grows with needs

Follow the phases in order, test thoroughly at each step, and you'll have a working system in 8 weeks.

**Remember:** This is a planning document. No code is implemented yet. Start with Phase 1 and build step by step.

---

**Document Version:** 1.0
**Last Updated:** October 30, 2026
**Author:** RFID Mobile Alerts Project

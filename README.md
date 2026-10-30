# RFID Mobile Alerts

Android mobile application and IoT backend for RFID door-lock access notifications.

## Overview

This project adds real-time mobile notifications to an existing RFID door-lock system. When someone scans an RFID card at the door, the system sends an instant notification to an Android phone with details about who entered and when.

## Current Status

**📋 Planning Phase**

This repository is currently in the planning stage. See [RFID_MOBILE_ALERTS_PLAN.md](RFID_MOBILE_ALERTS_PLAN.md) for the complete architecture and development roadmap.

## System Components

- **IoT Device:** ESP32 microcontroller with EM-18 RFID reader, LCD display, and servo motor
- **Backend:** Google Firebase (Firestore database + Cloud Functions)
- **Mobile App:** Android application built with Kotlin and Jetpack Compose
- **Communication:** Wi-Fi connectivity and Firebase Cloud Messaging (FCM)

## Key Features

- ✅ Real-time push notifications for door access events
- ✅ Authorized vs. unauthorized access alerts
- ✅ Access history with timestamps
- ✅ Offline operation (door works locally without internet)
- ✅ Multi-device notification support
- ✅ Secure cloud-based logging

## Hardware Requirements

### Existing Hardware
- Arduino Uno (to be replaced with ESP32)
- EM-18 RFID Reader (125 kHz UART)
- 16x2 LCD Display
- Servo Motor (SG90/MG995)
- Power Supply

### Additional Hardware Required
- ESP32 development board (replaces Arduino Uno)
- Logic level shifter (if keeping Arduino)
- Jumper wires and breadboard
- USB cable for programming

## Technology Stack

### IoT (ESP32)
- C++ with Arduino Framework
- Wi-Fi connectivity
- UART serial communication
- Firebase REST API

### Backend (Firebase)
- Firestore Database (NoSQL)
- Cloud Functions (Node.js)
- Firebase Cloud Messaging (FCM)

### Mobile (Android)
- Kotlin
- Jetpack Compose (UI)
- MVVM Architecture
- Firebase SDK

## Development Roadmap

See [RFID_MOBILE_ALERTS_PLAN.md](RFID_MOBILE_ALERTS_PLAN.md) for detailed phases:

1. **Phase 1:** Hardware breadboarding and serial communication testing
2. **Phase 2:** Cloud connection and database logging
3. **Phase 3:** Android UI and FCM push notifications
4. **Phase 4:** Field testing, security hardening, and enclosure assembly

## Getting Started

1. Read the complete plan: [RFID_MOBILE_ALERTS_PLAN.md](RFID_MOBILE_ALERTS_PLAN.md)
2. Gather required hardware components
3. Set up development environment (Arduino IDE, Android Studio, Firebase)
4. Follow Phase 1 instructions to begin prototyping

## Project Structure

```
rfid-mobile-alerts/
├── android-app/          # Android application source code
├── backend/              # Firebase Cloud Functions
├── iot/                  # ESP32 firmware source code
├── docs/                 # Additional documentation
├── RFID_MOBILE_ALERTS_PLAN.md  # Complete architecture plan
├── README.md             # This file
└── .gitignore            # Git ignore rules
```

## Security Considerations

- Door lock works locally even without internet (failsafe)
- Firestore security rules restrict data access
- API keys and secrets never committed to Git
- HTTPS required for all cloud communications
- Physical security for IoT device enclosure

## License

This project is private and proprietary.

## Contact

For questions or support, please refer to the project documentation or contact the development team.

---

**Note:** This is a planning repository. No production code has been implemented yet. Refer to the planning document for implementation details.

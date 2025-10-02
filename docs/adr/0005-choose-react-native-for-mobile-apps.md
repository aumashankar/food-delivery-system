# 0005 – Choose React Native for Customer & Driver Mobile Apps
*Status:* Accepted  
*Deciders:* Umashankar  
*Date:* 2025-09-30  

## Context
We need fast delivery of **two apps** (Customer & Driver) with shared UI/business logic, strong maps integration, background location, and push notifications. Team has strong web/React skills.

## Decision Drivers
- Speed to market with shared codebase
- Mature ecosystem for maps, push, background tasks
- Hiring/skills alignment (JS/TS)
- Good enough performance for our use case

## Considered Options
- Native (Kotlin + Swift)  
- **React Native** (chosen)  
- Flutter  
- Hybrid webview (Cordova/Capacitor)

## Decision
Use **React Native** with TypeScript, leveraging:
- **Navigation, Forms**: react-navigation, react-hook-form  
- **Maps/Location**: react-native-maps / vendor SDKs; background geolocation for driver pings (2–5s)  
- **Notifications**: FCM/APNs (notifee or native modules)  
- **OTA updates**: CodePush/AppCenter where allowed  
- **Shared modules**: reuse validation/models with BFF (TypeScript)

## Consequences
- Pros: Faster delivery across two apps; large ecosystem; shared skills.  
- Cons: Native edge cases (battery/permissions/OEM quirks); occasional bridge performance tuning.


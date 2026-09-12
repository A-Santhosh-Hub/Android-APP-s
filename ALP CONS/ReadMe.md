# ALP CONS — Astro Talk

**Version:** 1.0.0-INTERNAL  
**Developer:** Santosh  
**Project Identity:** Professional Internal Consultation Management System

---

## 🌌 Overview
ALP CONS (Astro Talk) is a high-security enterprise application designed for astrology consultation management. The system features a proprietary **Sequential Allocation Engine** that ensures fair and efficient distribution of consultation opportunities among practitioners (Teachers, Trainers, and Coaches).

## 🚀 Key Features

### 🛠 Administrative Control
*   **Operations Dashboard**: Real-time visibility into the consultation pipeline (Available, Offered, Accepted, Active, History).
*   **Staff Directory**: Full CRUD management of internal users, including role assignment and activation/deactivation.
*   **Consultation Factory**: Granular creation of sessions with custom fees, scheduling, and staff-specific eligibility.
*   **Link Management**: Secure provisioning of Zoom meeting URLs for accepted sessions.

### 🧘 Practitioner Experience
*   **Reactive Dashboard**: Instant visibility of personalized offers and active sessions.
*   **Smart Offer System**: Dedicated interface for accepting or declining sequential offers with a live-synced countdown.
*   **One-Tap Join**: Integrated "Join Zoom Meeting" action that automatically updates session status.
*   **Private History**: Secure access to personal session records.

---

## ⚙️ Technical Architecture

The application is built on a **Modern Android Stack** following **Clean Architecture** and **MVVM** patterns:

*   **UI**: 100% Jetpack Compose with Material 3 ("Cosmic Indigo" theme).
*   **Language**: Idiomatic Kotlin with Coroutines and StateFlow.
*   **Backend**: Firebase (Authentication, Firestore, Cloud Messaging).
*   **Navigation**: Type-safe Jetpack Navigation with Deep Link support.
*   **Messaging**: Firebase Cloud Messaging (FCM) for high-priority service notifications.

---

## 🏗 The Sequential Allocation Engine

The heart of ALP CONS is its allocation logic, which prevents "grab-and-go" competition and ensures quality:

1.  **Initiation**: Admin defines a list of eligible candidates and a **Notification Delay** (e.g., 10s, 30s).
2.  **Rotation (T+Delay)**: The offer is sent to Candidate A. If they decline or the timer expires, the engine atomically advances to Candidate B.
3.  **Atomic Locking**: Using **Firestore Transactions**, the system ensures that the moment a practitioner clicks "ACCEPT," the consultation is locked. Any simultaneous attempts by other users are rejected with a "Consultation already taken" state.
4.  **Watchdog**: A background monitoring system ensures the sequence advances even if practitioners are offline.

---

## 🔄 Consultation Lifecycle

| Status | Description |
| :--- | :--- |
| **AVAILABLE** | Created by Admin, awaiting allocation initiation. |
| **OFFERED** | Currently rotating through the eligible staff sequence. |
| **ACCEPTED** | Claimed by a practitioner; awaiting meeting link. |
| **WAITING FOR MEETING** | Link provided by Admin; ready to start. |
| **IN_PROGRESS** | Practitioner has joined the meeting. |
| **COMPLETED** | Session finished and settled. |
| **CANCELLED / EXPIRED** | Invalidated by Admin or sequence exhausted without claim. |

---

## 🛡 Security & Permissions

The system underwent a rigorous **Phase 17 Security Audit**:
*   **Role-Based Access Control (RBAC)**: Enforced via hardened Firestore Security Rules.
*   **Field Immutability**: Critical fields like `amount` and `eligibleRole` cannot be modified by staff.
*   **Meeting Privacy**: Meeting links are stored in isolated sub-collections, readable *only* by the Admin and the assigned practitioner.
*   **Obfuscation**: Release builds are protected by ProGuard to prevent reverse engineering.

---

## 📦 Deployment Checklist
1.  Verify `google-services.json` in `app/`.
2.  Deploy **Security Rules** via Firebase CLI.
3.  Create the initial `ADMIN` user manually in the Firestore `users` collection.
4.  Distribute the **Release APK** to internal staff.
5.  Staff must grant **Notification Permissions** on first launch to receive offers.

---
**Developed by Santosh**

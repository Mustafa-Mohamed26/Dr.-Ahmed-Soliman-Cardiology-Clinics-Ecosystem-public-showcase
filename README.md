# Dr. Ahmed Soliman Cardiology Clinics — Public Showcase

<p align="center">
  <img src="assets/Logo_README.png" alt="Dr. Ahmed Soliman Cardiology Clinics" width="420" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Flutter-%2302569B.svg?style=flat-square&logo=Flutter&logoColor=white" alt="Flutter" />
  <img src="https://img.shields.io/badge/Firebase-%23039BE5.svg?style=flat-square&logo=firebase" alt="Firebase" />
  <img src="https://img.shields.io/badge/TypeScript-%23007ACC.svg?style=flat-square&logo=typescript&logoColor=white" alt="TypeScript" />
  <img src="https://img.shields.io/badge/Gemini_AI-4285F4?style=flat-square&logo=google&logoColor=white" alt="Gemini AI" />
  <img src="https://img.shields.io/badge/WhatsApp-25D366?style=flat-square&logo=whatsapp&logoColor=white" alt="WhatsApp" />
</p>

> A **code showcase** demonstrating the architecture patterns used in the Dr. Ahmed Soliman Cardiology Clinics Ecosystem — a full-stack clinic management system powered by Flutter + Firebase Cloud Functions + Gemini AI.

---

## 📁 What Is in This Folder

| File | Description |
|------|-------------|
| `README.md` | This file — project overview |
| `CLEAN_ARCHITECTURE.md` | Clean Architecture deep-dive with real code examples |
| `WHATSAPP_FIREBASE_GEMINI.md` | How the WhatsApp webhook connects to Firebase + Gemini AI |
| `ARCHITECTURE.md` | End-to-end system architecture (sanitized) |
| `SECURITY.md` | Security model overview |

---

## 🌟 System Overview

A fully integrated **cardiology clinic management ecosystem** that unifies:

- **Staff Dashboard** — a Flutter (web) app for physician and clinic assistants.
  Schedule, unified inbox, bookings, digitization, patient records, AI assistant actions, emergency alerts, staff management.

- **WhatsApp AI Agent** — patients message the clinic on WhatsApp; a Gemini-powered LLM agent handles booking, waitlisting, FAQs, and emergency escalation automatically in Arabic and English.

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Flutter 3.x / Dart, BLoC/Cubit, injectable + get_it |
| Backend | Firebase Cloud Functions Gen 2 (TypeScript) |
| Database | Cloud Firestore |
| AI | Google Gemini with function-calling tools |
| Messaging | Meta WhatsApp Business Cloud API |
| Auth | Firebase Authentication (custom role claims) |
| Storage | Firebase Cloud Storage (PHI-gated signed URLs) |

---

## ✨ Key Features

### Staff Dashboard
- 📊 Live KPI overview
- 🗓️ Daily schedule with conflict detection
- 💬 Unified WhatsApp inbox (real-time Firestore streams)
- 📅 Booking management (create / reschedule / cancel)
- 🤖 AI assistant action review panel
- 🚨 Emergency alert system
- 🔎 Global search across patients, bookings, conversations

### WhatsApp AI Agent
- Conversational booking / cancellation / waitlist (Arabic + English)
- Guardrailed FAQ answers (no hallucinated services or prices)
- Automatic emergency detection → staff red alert + ER redirect
- Human handoff: AI pauses when a staff member replies
- Webhook signature validation (HMAC-SHA256)
- Voice note transcription (Gemini audio)

---

## 📸 Screenshots

<div align="center">

### Dashboard & Schedule

| Dashboard | Schedule |
|-----------|---------|
| <img src="assets/Dashboard.png" width="480"/> | <img src="assets/Schedule.png" width="480"/> |

### Unified Inbox

<img src="assets/Inbox.png" width="960"/>

### Patient Records & Digitization

| Patient Records | Digitization |
|----------------|-------------|
| <img src="assets/Patient_Records.png" width="480"/> | <img src="assets/Digitization.png" width="480"/> |

### AI Assistant Actions & Global Search

| AI Assistant Actions | Global Search |
|---------------------|--------------|
| <img src="assets/Ai_Assistant_Actions.png" width="480"/> | <img src="assets/Global_Search.png" width="480"/> |

</div>

---

## 📐 Architecture Patterns

See `CLEAN_ARCHITECTURE.md` for full code examples of:

1. **Inbox Feature** — Flutter Clean Architecture (Entity → UseCase → Repository → DataSource → Cubit)
2. **AI Orchestrator** — Backend pipeline pattern (multi-step coordinator with single-responsibility modules)

See `WHATSAPP_FIREBASE_GEMINI.md` for:

- WhatsApp webhook setup and signature verification
- Inbound message pipeline flow
- Gemini function-calling integration
- Firestore write-back pattern

---

## 📝 Note

This showcase contains **no production credentials, no patient data, no API keys, and no Firebase project IDs**.
All secrets are managed via Firebase Secret Manager.

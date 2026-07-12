# 🛰️ ShaloTrack — GPS Tracking & Fleet Management Platform

**Building a complete enterprise-grade GPS Tracking and Fleet Management platform from the ground up — from raw TCP socket bytes to a production-ready commercial system.**

![Status](https://img.shields.io/badge/status-active%20development-yellow?style=for-the-badge)
![Stack](https://img.shields.io/badge/stack-.NET%20%7C%20Python%20%7C%20Kotlin%20%7C%20Laravel-blue?style=for-the-badge)
![Database](https://img.shields.io/badge/database-PostgreSQL%20(Supabase)-336791?style=for-the-badge&logo=postgresql&logoColor=white)

---

## 📑 Table of Contents

1. [Introduction](#1-introduction)
2. [Company Background](#2-company-background)
3. [Project Journey](#3-project-journey)
4. [GPS Technology Overview](#4-gps-technology-overview)
5. [Hardware](#5-hardware)
6. [Communication Protocol (GT06 / V5)](#6-communication-protocol-gt06--v5)
7. [The Gateway](#7-the-gateway)
8. [The Parser](#8-the-parser)
9. [Backend API](#9-backend-api)
10. [Android Application](#10-android-application)
11. [Admin Portal](#11-admin-portal)
12. [Database Design](#12-database-design)
13. [Cloud Infrastructure](#13-cloud-infrastructure)
14. [Problems Solved](#14-problems-solved)
15. [Engineering Decisions](#15-engineering-decisions)
16. [Development Timeline](#16-development-timeline)
17. [Current Status](#17-current-status)
18. [Future Roadmap](#18-future-roadmap)
19. [Lessons Learned](#19-lessons-learned)
20. [Repository Structure](#20-repository-structure)
21. [Architecture Diagrams](#21-architecture-diagrams)
22. [Screenshots](#22-screenshots)
23. [API Documentation](#23-api-documentation)
24. [Deployment Guide](#24-deployment-guide)
25. [Contributing](#25-contributing)
26. [License](#26-license)

---

## 1. Introduction

**ShaloTrack** is a full-stack GPS Tracking and Fleet Management platform being built to replace legacy, closed-source GPS tracking systems with a modern, cloud-native, and scalable alternative.

Most commercial GPS tracking software in the market today is built on outdated architecture — desktop-era admin panels, unmaintained device protocols, and no real API layer for integration. ShaloTrack was started to solve that problem: build a platform that can communicate directly with physical GPS hardware over raw TCP sockets, process that data in real time, and expose it through a modern API to a mobile app, an admin/dealer portal, and (eventually) third-party integrations.

**Why GPS tracking systems are hard:**
- Devices communicate over unreliable 2G/GPRS networks, not clean REST APIs.
- Data arrives as raw binary packets, not JSON.
- The system has to handle thousands of concurrent, long-lived TCP connections.
- Vehicle events (ignition, alarms, engine cut) need near real-time processing.
- The backend has to reconcile "business" data (customers, subscriptions, vehicles) with high-frequency "telemetry" data (GPS points, heartbeats, raw packets) — two very different data access patterns living in the same platform.

ShaloTrack exists to solve all of this as one coherent, production-grade system rather than a collection of disconnected scripts.

## 2. Company Background

The project began as **Let'sTrack Lanka**, an industry project undertaken through LNBTI in partnership with a Sri Lankan GPS tracking technology provider. What started as an academic industry placement has since evolved into a **commercial product under the ShaloTrack Sri Lanka (Pvt) Ltd** brand.

- **Original name:** Let'sTrack Lanka (university industry project)
- **Current name:** ShaloTrack Sri Lanka
- **Why rebranded:** The project moved from a purely academic scope into an actual commercial deployment, with a real company, real GPS hardware, and a real (growing) customer base — so it needed a distinct commercial identity separate from the university project name.
- **Commercial goals:** Replace third-party GPS tracking software currently used by the company with an in-house platform that the business fully owns, controls, and can extend (fuel monitoring, dashcams, fleet analytics, driver behaviour) without being limited by a vendor's roadmap.

Unlike a typical university project, ShaloTrack is being developed **in collaboration with a real GPS tracking company**, using actual GPS tracker hardware (GT06 / V5 devices), production-grade communication protocols, and real operational workflows — installation, activation, subscriptions, and support.

## 3. Project Journey

> *"When this project began, I knew almost nothing about how commercial GPS tracking systems actually worked."*

Before writing a single line of production code, I spent weeks understanding how commercial GPS tracking systems function end-to-end — not just "show a pin on a map," but the full pipeline from satellite to phone screen.

This meant learning, largely from scratch:

- GPS satellite positioning fundamentals
- GSM/GPRS communication and how trackers get online
- TCP socket programming and connection lifecycle management
- Binary protocol design and packet parsing
- CRC (Cyclic Redundancy Check) validation
- GPS tracker firmware behaviour (login, heartbeat, alarms, commands)
- Vehicle ignition and power systems
- SMS-based device configuration
- Existing enterprise GPS platforms, to understand what "good" looks like

Rather than treating GPS trackers as black boxes that "just send location," I wanted to understand exactly how every byte travels — from the tracker installed in a vehicle's dashboard, across a GSM network, into a TCP listener, through a parser, into a database, and finally out through an API to a phone screen.

That decision — to understand the protocol at the byte level instead of relying on a third-party SDK — became the foundation the entire platform is built on.

## 4. GPS Technology Overview

At a high level, a single GPS coordinate makes this journey before it ever reaches a user's phone:

```
GPS Satellites
      │
      ▼
GPS Tracker (hardware, installed in vehicle)
      │  (GSM / GPRS)
      ▼
Mobile Network / SIM
      │  (raw TCP, binary packets)
      ▼
Internet
      │
      ▼
ShaloTrack Gateway (TCP Listener)
      │
      ▼
Packet Parser (GT06 / V5 protocol)
      │
      ▼
PostgreSQL Database (Supabase)
      │
      ▼
Backend API (ASP.NET Core)
      │
      ▼
Android App / Admin Portal
```

Each layer has its own responsibilities and failure modes:
- The **tracker** determines its own position via GPS satellites and packages it as a binary frame.
- The **network layer** (GSM/GPRS/TCP) is inherently unreliable — connections drop, packets can arrive out of order or split across TCP segments.
- The **Gateway** has to maintain potentially thousands of simultaneous long-lived TCP connections and know which socket belongs to which device (by IMEI).
- The **Parser** turns raw bytes into meaningful domain events (location update, heartbeat, alarm, ignition change).
- The **Database** must handle both slow-changing business data and extremely high-frequency telemetry writes.
- The **API and clients** need near real-time visibility into current vehicle status without querying the entire tracking history table.

## 5. Hardware

To build a correct parser and API, it wasn't enough to read protocol documentation — I also studied the physical hardware being installed in vehicles:

- **GT06 / V5 GPS trackers** — the two tracker models the platform currently supports
- **GPS module** — determines position from satellites
- **GSM module** — handles SIM-based network communication
- **SIM card provisioning** — data plans, APN configuration
- **Internal antennas** — GPS and GSM signal reception
- **ACC (ignition) detection wire** — tells the device when the vehicle is on/off
- **Relay module** — enables remote engine cut/immobilization
- **Power backup battery** — keeps the device reporting briefly if main power is cut (theft scenario)
- **Shock and motion sensors** — used for basic alarm/anti-theft events
- **Vehicle wiring** — how the device is physically installed and powered

Understanding the hardware side meant that when a packet said "ACC ON" or "low battery," I could interpret it correctly against real device behaviour — instead of guessing what a raw byte value "probably" meant.

## 6. Communication Protocol (GT06 / V5)

The heart of the platform's backend is a **custom binary protocol parser** built against the GT06/V5 GPS communication protocol specification.

**Packet structure understood and implemented:**

| Field | Purpose |
|---|---|
| Start Bits (`0x7878` / `0x7979`) | Marks the beginning of a packet, and its length format (normal vs extended) |
| Packet Length | Total length of the packet body |
| Protocol Number | Identifies the packet type (login, location, heartbeat, alarm, etc.) |
| GPS Information | Latitude, longitude, speed, course, satellite count |
| LBS Information | Cell tower / base station data, used as a location fallback |
| Terminal / Status Information | Ignition, voltage, GSM signal strength, alarm flags |
| Serial Number | Sequential packet counter, used for matching ACKs |
| CRC | Checksum used to validate packet integrity |
| Stop Bits (`0x0D0A`) | Marks the end of a packet |

**Packet types implemented:**
- **Login packet** — device authenticates using its IMEI number
- **Heartbeat packet** — periodic "I'm still alive" signal
- **Location packet** — GPS coordinates, speed, heading, timestamp
- **Status packet** — ignition state, voltage, GSM signal
- **Alarm packet** — SOS, power cut, vibration/shock, geofence, etc.
- **Command / ACK packets** — server-to-device commands (e.g., engine cut) and device acknowledgements

**Key protocol behaviours handled:**
- Devices must be acknowledged correctly, or they will retransmit or disconnect
- Extended-length packets (`0x7979`) needed separate length-parsing logic from standard packets (`0x7878`)
- Latitude/longitude hemisphere bits needed correct interpretation, otherwise coordinates land in the wrong hemisphere entirely
- TCP is a stream protocol, not a message protocol — packets can arrive split across multiple TCP reads or multiple packets can arrive in a single read, so the parser needs proper buffering and framing logic, not naive "one read = one packet" assumptions
- CRC validation is used to reject corrupted packets before they ever touch the database

## 7. The Gateway

The Gateway is a **custom Python TCP service** — not a third-party IoT platform — built specifically to handle GPS tracker connections at scale.

```
GPS Tracker
     │  (TCP connect)
     ▼
TCP Listener  ──────►  Socket Registry  (maps IMEI → active socket)
     │
     ▼
Packet Router  (splits/reassembles raw TCP stream into complete packets)
     │
     ▼
Parser  (decodes packet type, validates CRC)
     │
     ▼
Repository Layer  (writes telemetry to the database)
     │
     ▼
Command Service  (looks up the device's active socket to send commands, e.g. engine cut)
```

**Responsibilities of the Gateway:**
- Accept and hold thousands of simultaneous long-lived TCP connections
- Identify each connection by the device's IMEI after login
- Maintain a live **socket registry** so the API layer can push commands (like "cut engine") down to a *specific, currently connected* device
- Reassemble TCP stream data into complete, valid packets before handing off to the parser
- Handle disconnects and reconnects gracefully without losing device state
- Forward decoded events to the persistence layer in near real time

This is one of the more technically demanding parts of the system, because it behaves less like a typical web backend and more like an infrastructure-level networking service.

## 8. The Parser

Sitting just after the Gateway's packet router, the parser layer is responsible for turning validated binary packets into meaningful domain objects.

Implemented parsers include:
- **Login parser** — extracts and validates IMEI, confirms device identity
- **Location parser** — decodes latitude, longitude, speed, course, satellite count, timestamp
- **Heartbeat parser** — confirms device liveness, updates last-seen state
- **Alarm parser** — decodes alarm type (SOS, power cut, vibration, geofence) and severity
- **Status/Information parser** — decodes ignition state, voltage level, GSM signal strength
- **Command response parser** — decodes device acknowledgement of server-issued commands

Each parser follows the same pipeline: **decode → validate CRC → map to a strongly typed domain model → hand off to the repository layer → build and send the correct ACK back to the device.** Getting the ACK format wrong causes devices to endlessly retransmit or silently drop the connection — this was one of the trickiest classes of bugs to track down (see [Problems Solved](#14-problems-solved)).

## 9. Backend API

The core business API is built with **ASP.NET Core Web API**, following clean architecture principles.

**Stack:**
- ASP.NET Core Web API
- Entity Framework Core
- PostgreSQL (via Supabase)
- JWT Authentication + Firebase Authentication
- Repository Pattern + Unit of Work
- Dependency Injection throughout
- Swagger/OpenAPI documentation

**Architectural evolution:** the API originally started as a straightforward CRUD service. As the platform matured, it split cleanly into two domains with very different access patterns:

- **Business Domain** — Customers, Vehicles, GPS Devices, Device Assignments, Dealers, Subscriptions. Relatively low write-frequency, standard CRUD + business rules.
- **Telemetry Domain** — Current Locations, GPS Tracking History, Device Events, Raw Packet Logs. Extremely high write-frequency (constant inbound data from the Gateway), and read patterns optimized around "give me the latest known state," not full history scans.

To keep telemetry reads fast, the API uses **CQRS-inspired projection queries** — read-only, purpose-built query models (e.g. a lightweight "current location" projection) instead of materializing full entity graphs for every request.

**Current backend functionality:**
- Customer management
- Vehicle management
- GPS device registration and IMEI validation
- Device-to-vehicle assignment
- Subscription management
- Dealer management
- Authentication & role-based authorization
- Current-location and tracking-history endpoints
- Swagger-documented REST API

## 10. Android Application

The customer-facing mobile app is built in **Kotlin**, following modern Android architecture practices.

**Implemented / in progress:**
- Firebase Authentication (Email + Phone OTP)
- Google Maps integration for live vehicle position
- Dashboard with vehicle summary
- Vehicle list and selection
- Live tracking view
- User registration and login flow
- Profile management

**Planned:** trip history playback, push notifications for alerts, and remote command controls (e.g. engine cut) surfaced directly in the app.

## 11. Admin Portal

The internal/dealer-facing administration portal is being built with **Laravel**, replacing an earlier plain-PHP admin prototype.

Planned and in-progress capabilities:
- Role-based access control (Admin / Dealer / Support)
- Customer and fleet management
- Device management and assignment
- Subscription and billing oversight
- Reporting and basic analytics
- ERP-style operational workflows for the business side of ShaloTrack

## 12. Database Design

The platform uses **PostgreSQL**, hosted via **Supabase**, with a schema deliberately normalized to support both business data and high-frequency telemetry without the two colliding.

**Core tables (by domain):**

*Business domain:*
- Customers
- Vehicles
- Dealers
- Subscriptions
- Device Assignments

*Telemetry domain:*
- GPS Devices
- Tracking History
- Current Locations
- Trips
- Alerts / Events
- Raw Packet Logs

*Platform:*
- Geofences
- Audit Logs

The split between business tables and telemetry tables mirrors the API's domain split — it allows the tracking/history tables (which grow very quickly) to be indexed and queried differently from the comparatively stable business tables, without over-normalizing the whole schema into something slow to query in practice.

## 13. Cloud Infrastructure

```
Internet
   │
   ▼
Cloudflare (edge / DNS / admin portal hosting)
   │
   ├──► ASP.NET Core API  (Amazon EC2)
   │           │
   │           ▼
   │      Supabase PostgreSQL
   │
   ├──► Python Gateway (TCP, Amazon EC2)
   │           │
   │           ▼
   │      Supabase PostgreSQL
   │
   ├──► Firebase (Auth + Push Notifications)
   │
   └──► Google Maps Platform (map rendering, geocoding)

Clients: Android App  ·  Laravel Admin Portal
```

**Stack:**
- Amazon EC2 for API and Gateway hosting
- Docker for containerization (planned rollout across services)
- Supabase-managed PostgreSQL
- GitHub Actions for CI/CD (planned)
- Cloudflare for the admin portal / static hosting and edge protection
- Firebase for authentication and push notifications
- Google Maps Platform for map rendering

## 14. Problems Solved

A selection of real engineering problems encountered and resolved during development:

- **ACK format bugs** — incorrect acknowledgement structure caused devices to endlessly retransmit or drop connections; fixed by matching the ACK format exactly to the protocol spec per packet type.
- **`0x7979` extended packets** — some packets use a different start-bit/length format than standard `0x7878` packets; required separate length-parsing logic to avoid truncating or misreading packets.
- **Latitude/longitude hemisphere bits** — misreading the hemisphere flag silently placed vehicles in the wrong hemisphere; fixed by correctly decoding the sign/hemisphere bit per the protocol spec.
- **TCP packet splitting/merging** — TCP is a stream, not a message protocol; a naive "one socket read = one packet" assumption caused corrupted parses. Solved with proper buffering and packet framing in the Gateway's packet router.
- **Socket registry consistency** — keeping an accurate live map of IMEI → active socket connection, especially across reconnects, so outbound commands reach the correct, currently-connected device.
- **Unknown/unregistered IMEI handling** — devices reporting in before being registered in the business database required a safe "hold and log" path instead of dropping data or crashing the pipeline.
- **Database write pressure from telemetry** — high-frequency location/heartbeat writes required rethinking indexing and introducing a "current location" projection table instead of always querying full history.
- **API performance under telemetry load** — separating business and telemetry query paths (see [Backend API](#9-backend-api)) to keep business endpoints fast regardless of tracking data volume.

## 15. Engineering Decisions

Key technology choices, and the reasoning behind them:

- **PostgreSQL over MySQL** — stronger support for complex queries, JSON columns for flexible event/alarm payloads, and better long-term scalability for a telemetry-heavy workload.
- **Python for the Gateway** — fast to iterate on while prototyping protocol parsing, with mature libraries for socket handling; performance-critical paths were kept lean and the design allows a future rewrite of hot paths if needed.
- **ASP.NET Core for the business API** — strong typing, mature ecosystem, first-class support for clean architecture patterns (repository, unit of work, DI), and good performance for a REST API layer.
- **Laravel for the Admin Portal** — fast to build CRUD-heavy, RBAC-driven admin interfaces without reinventing standard web-app scaffolding.
- **Supabase (managed PostgreSQL)** — removes the operational overhead of self-managing a database while still giving direct PostgreSQL access, useful during active development.
- **Amazon EC2 over fully managed PaaS** — needed direct control over long-lived raw TCP socket connections for the Gateway, which doesn't fit cleanly into typical serverless/PaaS hosting models.
- **Docker (in progress)** — consistent environments across development and production, and a clear path toward CI/CD-driven deployments.
- **GitHub Actions (planned)** — keeps CI/CD in the same ecosystem as source control, avoiding a separate CI platform to manage.

## 16. Development Timeline

```
May
 └─ Research: GPS technology, GSM/GPRS, GT06/V5 protocol study, requirements gathering

June
 ├─ Python Gateway (TCP listener, socket registry)
 ├─ GT06/V5 Parser implementation
 ├─ Database design and normalization
 └─ Full system/infrastructure/deployment architecture design

July
 ├─ ASP.NET Core API development (business + telemetry domains)
 ├─ Android application development (Firebase, Maps, dashboard)
 ├─ Laravel Admin Portal (initial build)
 └─ Cloud deployment planning (EC2, Docker, CI/CD)
```

*(This section is updated as the project progresses.)*

## 17. Current Status

| Component | Status |
|---|---|
| GPS Gateway (TCP + Socket Registry) | ✅ Complete |
| GT06 / V5 Protocol Parser | ✅ Complete |
| Database Design | ✅ Complete |
| Backend API | 🚧 In Progress |
| Android Application | 🚧 In Progress |
| Admin Portal (Laravel) | 🚧 In Progress |
| Cloud Deployment (EC2 / Docker / CI-CD) | 🟡 Planned |
| Automated Testing | 🟡 Planned |

## 18. Future Roadmap

**v0.4 — Telemetry APIs**
- Current location endpoints
- GPS tracking history endpoints
- Device status endpoints
- Event/alarm endpoints
- Raw packet inspection tooling

**v0.5 — Admin Portal**
- Reports and analytics
- Full RBAC
- ERP-style operational workflows

**v0.6 — Android Features**
- Live tracking polish
- Trip playback
- Push notifications
- Remote commands (engine cut, etc.) from the app

**v1.0 — Production Release**
- Full commercial deployment for ShaloTrack customers
- Fleet analytics
- Fuel monitoring
- Driver behaviour analysis
- Geofencing
- Dashcam integration
- Payment gateway integration
- iOS application
- 4G GPS device support
- AI-powered fleet insights

## 19. Lessons Learned

This project expanded my understanding of software engineering well beyond typical application development:

- **Distributed systems** — coordinating state across a Gateway, database, and API that all need a consistent view of "what is this device doing right now."
- **TCP/IP networking** — the difference between a message-oriented mental model and how a stream socket actually behaves.
- **IoT and embedded communication** — reasoning about hardware constraints (power, connectivity, firmware behaviour) when designing backend logic.
- **Binary protocol parsing** — working directly with bytes, bit flags, and checksums instead of JSON.
- **Cloud and infrastructure design** — choosing hosting models based on actual connection/workload characteristics, not just convention.
- **API design at scale** — separating access patterns (business vs. telemetry) instead of forcing one data model to serve two very different workloads.
- **Production mindset** — building for a real company with real customers changes the bar significantly compared to a purely academic project.

## 20. Repository Structure

```
ShaloTrack/
├── shalotrack-gateway/      # Python TCP Gateway + GT06/V5 parser
├── shalotrack-api/          # ASP.NET Core business & telemetry API
├── shalotrack-admin/        # Laravel admin/dealer portal
├── shalotrack-mobile/       # Kotlin Android application
├── shalotrack-docs/         # Architecture docs, ERDs, diagrams
└── shalotrack-website/      # Public-facing marketing site
```

## 21. Architecture Diagrams

> Diagrams to be added: System Architecture, Infrastructure Architecture, Deployment Architecture, Database ERD, Gateway Sequence Diagram, Parser Flow Diagram, API Layered Architecture, Use Case Diagram, Activity Diagrams, Class Diagrams.

## 22. Screenshots

> Screenshots to be added: Android app (dashboard, live tracking), Swagger API docs, Admin Portal, Gateway logs/console, Database schema view.

## 23. API Documentation

> Full endpoint reference (routes, request/response examples, auth requirements, business rules) to be published here or linked to the Swagger/OpenAPI spec once the business and telemetry APIs stabilize.

## 24. Deployment Guide

> Step-by-step deployment instructions (EC2 setup, Docker build/run, environment variables, database migration, Gateway startup) to be documented once the CI/CD pipeline is finalized.

## 25. Contributing

This is currently a closed commercial project under active development for ShaloTrack Sri Lanka. Contribution guidelines will be published if/when the project opens to external collaborators.

## 26. License

> License to be determined — this repository currently reflects proprietary work for ShaloTrack Sri Lanka (Pvt) Ltd.

---

<p align="center"><i>🚧 Status: Active Development — this repository evolves continuously as ShaloTrack moves toward a production-ready commercial platform.</i></p>

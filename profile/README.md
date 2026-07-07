# Smart Palm Fertilization Tractor Automation

Autonomous tractor system for precision fertilization in oil palm plantations with GPS navigation, computer vision, depth sensing, and RFID-based per-tree nutrient dosing.

**Tags:** Computer Vision · Edge AI · RFID · IoT · GPS

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Architecture Diagram](#2-architecture-diagram)
3. [How the System Operates](#3-how-the-system-operates)
4. [Architecture Breakdown](#4-architecture-breakdown)
5. [Technology Stack](#5-technology-stack)
6. [Field Documentation](#6-field-documentation)

---

## 1. Abstract

Conventional palm oil fertilization relies on manual driving and uniform dosing, so fertilizer is often over-applied or mis-targeted because each tree has different nutrient needs.

This system automates the full cycle on a moving tractor: GPS follows a mapped route, computer vision detects palm trees, a depth camera confirms positioning distance, RFID reads per-tree nutrient data, and a sprayer applies the calculated dose. Telemetry and fertilization events sync to a remote backend via Starlink for remote monitoring and historical records.

---

## 2. Architecture Diagram

Full system view: on-tractor edge stack (left), business logic flow (center), server infrastructure (right). Connectivity between field and server uses Starlink satellite internet.

<img width="1693" height="929" alt="tractor-architecture" src="https://github.com/user-attachments/assets/f2367e7e-a5ff-45a1-9ab6-e93998834574" />


*Smart Palm Oil Fertilization Autonomous Tractor System Architecture*

**Legend**

| Indicator | Meaning |
|-----------|---------|
| Blue | Data flow |
| Green | Control signal |
| Cyan | Network / communication |

---

## 3. How the System Operates

Each fertilization run follows a fixed 10-step sequence (center column of the architecture diagram above):

| Step | Module | Description |
|------|--------|-------------|
| 01 | **GPS positioning** | Tractor location is read and sent to the navigation module. |
| 02 | **Route following** | Path planning module guides the tractor along a predefined operational route. |
| 03 | **Object detection** | Forward camera detects objects in the tractor's path. |
| 04 | **Tree classification** | YOLO11 identifies whether the detected object is an oil palm tree. |
| 05 | **Distance measurement** | Intel RealSense depth camera measures tractor-to-tree distance (e.g. 2 m threshold). |
| 06 | **RFID read** | UHF/MFRC522 reader retrieves tree ID and stored plant data from the tag. |
| 07 | **Nutrient lookup** | Data service fetches nutrient requirements and fertilization history for that tree. |
| 08 | **Dose calculation** | Dosage engine computes the required fertilizer amount. |
| 09 | **Sprayer activation** | Arduino microcontroller triggers the fertilizer tank and sprayer system. |
| 10 | **Event logging** | Fertilization event is recorded locally and synced to the backend. |

---

## 4. Architecture Breakdown

Each block in the architecture diagram maps to a layer below. Data flows from on-tractor hardware through edge processing, then over Starlink to the remote backend, database, and user interfaces.

### Layer 1 - Hardware Layer (On Tractor)

All physical components mounted on the autonomous tractor. Sensor data is collected locally; actuation is handled through a separate microcontroller to keep real-time control isolated from the main compute unit.

**Sensors**

- **Intel RealSense depth camera** - captures depth frames for object detection and distance measurement between the tractor and palm trees ahead
- **GPS module** - provides latitude/longitude for navigation and route tracking along predefined operational paths
- **RFID reader (MFRC522 / UHF)** - reads tags attached to each palm tree; tags store tree ID, nutrient requirements, and fertilization history

**Actuators**

- **Fertilizer tank & sprayer system** - holds liquid fertilizer and applies it at a controlled rate when triggered
- **Arduino microcontroller** - receives control signals from the Jetson and drives the sprayer relay; keeps actuator timing independent of the main OS

**On-board computing**

**NVIDIA Jetson Orin Nano** acts as the edge computing hub. It ingests camera, GPS, and RFID data, runs AI inference and business logic, and sends control commands to the Arduino. All three sensor streams converge here before any remote upload.

### Layer 2 - Edge AI & Processing Layer

Software running on the Jetson Orin Nano. Handles all time-sensitive decisions in the field (detection, distance checks, dosage calculation) without waiting for server round-trips.

**Computer vision pipeline**

Depth frames from the RealSense camera are fed into **YOLO11 Nano**, optimized with **NCNN** for inference on the Jetson. The model classifies objects ahead as palm trees or non-targets, triggering the distance measurement step only when a tree is confirmed.

**Core services (on edge)**

- **Sensor fusion** - combines GPS position, camera detections, and RFID reads into a single operational context
- **Distance estimation** - uses depth data to measure tractor-to-tree distance; spraying is armed when the distance matches the configured threshold (e.g. 2 m)
- **Object classification** - filters detections to oil palm trees only
- **RFID data processing** - parses tag data and links it to the tree record in local cache or backend
- **Fertilizer dosage calculation** - computes application volume based on per-tree nutrient requirements
- **Event logging** - records each fertilization event with timestamp, tree ID, dose applied, and GPS coordinates

**On-device AI assistant**

**Ollama + Gemma** runs locally on the Jetson for operational support, including diagnostics, decision assistance, and natural language queries from field operators without requiring internet access.

### Layer 3 - Business Logic Flow

The operational sequence that ties all edge modules together. Each step maps to a dedicated module; the full cycle repeats for every tree along the route.

- **Navigation module** - reads GPS position and feeds it to path planning
- **Path planning module** - guides the tractor along the predefined route mapped before the run
- **Vision system** - camera detects objects in the tractor's forward path
- **Detection module** - YOLO11 confirms the object is a palm tree
- **Distance estimation** - depth camera verifies the tractor is within spraying range
- **RFID module** - reads the tree tag and retrieves stored plant data
- **Data service** - fetches nutrient requirements and past fertilization records for that tree
- **Dose calculator** - determines the fertilizer amount based on tree-specific needs
- **Actuator control** - sends the spray command to Arduino
- **Logging module** - writes the event to local storage and queues it for remote sync

Steps 1–10 run entirely on the edge device. Remote sync happens asynchronously after step 10.

### Layer 4 - Backend Infrastructure Layer

Hosted **Express.js REST API server** that receives tractor telemetry, fertilization events, and configuration requests over Starlink (or 4G/Wi-Fi when available).

- **API gateway** - single entry point for all client and tractor requests
- **Authentication & authorization** - secures access for dashboards, desktop apps, and tractor clients
- **Business logic** - route management, tree registry, fertilization scheduling, and analytics aggregation
- **Data validation** - validates incoming telemetry and event payloads before database writes
- **WebSocket** - pushes real-time tractor status, telemetry, and alerts to connected dashboards
- **File & log management** - stores system logs, AI assistant transcripts, and uploaded configuration files

The backend sits in the hosted server environment and is the central coordination point between field devices and admin interfaces.

### Layer 5 - Database Layer

**MySQL** stores all persistent data accessed by the backend and dashboards.

- **Tree information** - RFID tag IDs, tree attributes, plantation block, and GPS coordinates
- **Nutrient requirements** - per-tree N/P/K dosing profiles and recommended application rates
- **Fertilization history** - timestamped records of every application: tree ID, dose, operator, and tractor ID
- **GPS routes** - predefined operational paths mapped before each fertilization run
- **Tractor telemetry** - position, speed, sensor status, and tank level streamed during operation
- **AI logs** - on-device assistant queries and responses for audit and debugging

Tree and nutrient data is written during plantation setup; fertilization history and telemetry accumulate during each operational run.

### Layer 6 - Application Layer (User Interface)

Two client applications connect to the backend via REST and WebSocket: one for remote administration, one for on-site field use.

**React.js web admin dashboard**

Browser-based panel for plantation managers. Displays a live map with tractor position and route, real-time telemetry gauges, and fertilization analytics. Supports route and tree management, remote configuration of dosing parameters, and interaction with the on-device AI assistant through the backend.

**PyQt5 desktop application**

Native desktop client with the same core features including fleet monitoring, route/tree management, analytics, remote configuration, and real-time tractor status. Used on-site where a dedicated workstation is available and browser access is less practical.

### Connectivity - Starlink: Field to Server

Oil palm plantations often lack reliable terrestrial internet. The on-tractor Jetson connects to a local router via LAN/Wi-Fi, which uplinks through **Starlink satellite internet** to the hosted backend server.

- **On-tractor client** - Jetson publishes telemetry and fertilization events; pulls route updates and configuration changes
- **Starlink uplink** - provides the field-to-server bridge where cellular coverage is unavailable
- **Backend server** - Express.js backend receives data and serves both React and PyQt5 clients

When Starlink or cellular is unavailable, edge processing and local logging continue; events sync once connectivity is restored.

---

## 5. Technology Stack

| Area | Technologies |
|------|--------------|
| **Edge & Vision** | Python · C++ · YOLO11 Nano (NCNN) · NVIDIA Jetson Orin Nano · Intel RealSense |
| **Hardware Control** | Arduino · MFRC522 / UHF RFID · GPS module · Fertilizer sprayer system |
| **On-Device AI** | Ollama · Gemma model |
| **Backend** | Express.js · MySQL · WebSockets |
| **Frontend** | React.js (web dashboard) · PyQt5 (desktop app) |
| **Connectivity** | Starlink · MQTT / REST |

---

## 6. Field Documentation

Photos and videos from field deployment and system testing.

### Videos


https://github.com/user-attachments/assets/03828fcf-77a3-4f05-aacb-f939496c2b60



https://github.com/user-attachments/assets/7015dbd9-a0b1-4dbc-abb7-99c2aab64600

https://github.com/user-attachments/assets/3827e6e6-752e-4893-a5f9-0cd35b2e7622



https://github.com/user-attachments/assets/3a4c3f06-640c-4c8d-9e69-b56ee2ced1b4

<img width="4032" height="3024" alt="2-h" src="https://github.com/user-attachments/assets/3d0f4661-3744-43ef-baed-ddfd5650ee5c" />


https://github.com/user-attachments/assets/d41bb7e9-f28a-4f17-81b3-1bc361b43ede



<img width="3024" height="4032" alt="1-v" src="https://github.com/user-attachments/assets/7e54da6d-4c07-4e65-a93a-f4f39d11a0a0" />

# Level 13 - Configuration

## Purpose
This document provides the technical specifications for the **Configuration System**. It defines the complete YAML schema for all configuration files, including data types, validation rules, and default values.

## Master Configuration Schema (`marengo.yaml`)
This schema, recovered from `deprecated/CONFIGURATION_SYSTEM.md`, defines the structure of the master `marengo.yaml` file. It includes sections for all major components and is designed to be hot-reloadable.

```yaml
# config/marengo.yaml

version: "1.0.0"
mode: "online_learning"  # online_learning | autonomous

# Blue Iris connection
blue_iris:
  host: "http://192.168.1.50"
  port: 81
  username: !secret bi_username
  password: !secret bi_password
  timeout_ms: 1000

# Frame acquisition
acquisition:
  frame_rate: 10.0
  cameras_concurrent: 20
  jpeg_quality: 85
  cpu_throttle:
    enabled: true
    cpu_threshold_percent: 80
    throttle_step: 0.5
    min_frame_rate: 1.0

# YOLO detection
detection:
  model_path: "models/yolov8s.pt"
  yolo_batch_size: 4
  yolo_threads: 4
  confidence_threshold: 0.5
  max_pending_frames: 50
  drop_strategy: "oldest"
  
  classes:
    person: {enabled: true, confidence: 0.6}
    vehicle: {enabled: true, confidence: 0.5}
    animal: {enabled: true, confidence: 0.65}

# Intra-camera tracking
tracking:
  max_tracks_per_camera: 50
  update_interval: 0.1
  iou_threshold: 0.3
  max_age_seconds: 3.0

# Cross-camera merging
merging:
  enabled: true
  auto_merge_threshold: 0.85         # Higher in online_learning mode
  human_review_queue: true           # Enabled in online_learning mode
  
# External AI services
external_ai:
  global_qps_limit: 20
  services:
    - name: "cpai-gpu-desktop"
      url: "http://192.168.1.100:32168"
      priority: 1
      enabled: true
      health_check_interval_s: 30
      models:
        face_recognition: {enabled: true, endpoint: "/v1/vision/face/recognize", max_qps: 10, timeout_ms: 300}
        lpr: {enabled: true, endpoint: "/v1/vision/lpr", max_qps: 5, timeout_ms: 400}
          
    - name: "cpai-cpu-fallback"
      url: "http://localhost:32168"
      priority: 2
      enabled: true
      health_check_interval_s: 30
      models:
        face_recognition: {enabled: true, endpoint: "/v1/vision/face/recognize", max_qps: 3, timeout_ms: 500}
        lpr: {enabled: false}

# Storage
storage:
  base_path: "C:/MarengoCam/storage"
  frames_retention_hours: 216  # 9 days
  metadata_retention: "indefinite"
  write_batch_interval: 5.0

# Per-camera overrides
cameras:
  FrontGate:
    enabled: true
    face_recognition:
      enabled: true
      min_confidence: 0.75
    lpr:
      enabled: true
      min_confidence: 0.70
      
  BackYard:
    enabled: true
    face_recognition:
      enabled: true
      min_confidence: 0.65
    lpr:
      enabled: false

# Hot-reload
hot_reload:
  enabled: true
  watch_interval_s: 5
  validation:
    enabled: true
    auto_rollback_on_errors: true

# Secrets (external file)
secrets_file: "config/secrets.yaml"
```

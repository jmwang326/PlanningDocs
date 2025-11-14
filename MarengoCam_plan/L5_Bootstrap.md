# Level 5 - Bootstrapping Strategy

## 1. Purpose
This document outlines the strategy for bootstrapping the MarengoCam system. Bootstrapping is the process of using historical or offline data to pre-populate the system's knowledge bases, such as the `6x6` inter-camera grids and the Re-ID model's training set.

A "cold start" with no historical knowledge would lead to poor initial performance and a long, slow learning period. This strategy aims to accelerate the system's path to high performance by giving it a foundational dataset from day one.

## 2. Strategic Goals

1.  **Populate the `6x6` Inter-Camera Grids:** The primary goal is to learn the most common travel times and overlap zones between all camera pairs. A well-populated grid is essential for the Track Merging Engine to function effectively.

2.  **Seed the Re-ID Model Training Set:** To achieve high-accuracy Re-ID, we must fine-tune a model on data specific to this environment. The bootstrapping process will be the primary source for this initial training data.

3.  **Establish Initial Agent Identities:** Identify recurring agents (people and vehicles) from historical footage to begin building out the `agents` and `identities` tables.

## 3. Approach

The bootstrapping process will be an offline, one-time (or periodic) operation, not part of the live system. It will consist of a dedicated script or tool that processes a large corpus of historical video footage.

### 3.1. Data Source
- **Input:** A curated set of historical video files (e.g., several weeks of 24/7 recordings).
- **Selection Criteria:** The footage should be representative of typical activity, including various lighting conditions and weather.

### 3.2. Processing Phases
The process will run in distinct, sequential phases to ensure data quality:

1.  **Phase 1: Track Generation:**
    - Process all historical footage through the YOLO detector and tracker to generate a complete set of `track` data for the entire period. This is a computationally intensive, one-time cost.

2.  **Phase 2: High-Confidence Merging:**
    - Run a "conservative" version of the merging logic on the generated tracks. This logic will only perform merges that are extremely high-confidence (e.g., tracks with significant temporal and spatial overlap). The goal here is accuracy, not completeness.

3.  **Phase 3: Knowledge Extraction & Database Seeding:**
    - **Grid Learning:** Use the high-confidence merges to populate the `inter_camera_adjacency` table with initial travel times.
    - **Re-ID Data Generation:** For each newly created `agent` with multiple validated tracks, extract high-quality image crops (tracklets). These tracklets will be organized and labeled, forming the initial dataset for fine-tuning the Re-ID model.

## 4. Status
This document is a placeholder and is considered **in-progress**. The detailed technical specification for the bootstrapping tools and processes will be defined in `L15_Bootstrap.md`.

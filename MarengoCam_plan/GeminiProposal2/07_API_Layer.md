# API Layer and Messaging

The API layer provides a unified interface for interacting with the MarengoCam system. This includes a RESTful API for external clients and the dashboard, as well as an MQTT-based messaging system for real-time, event-driven communication between internal components.

## RESTful API

The REST API will be the primary way for the dashboard and other clients to query data and initiate actions.

**Base URL**: `/api/v1`

### Endpoints

- **GET /api/v1/tracks**: Retrieve a list of all tracks.
  - Query Params: `camera_id`, `start_time`, `end_time`, `identity_id`
- **GET /api/v1/tracks/{track_id}**: Retrieve details for a specific track.
- **GET /api/v1/chunks**: Retrieve a list of processed chunks.
- **GET /api/v1/chunks/{chunk_id}**: Retrieve details for a specific chunk.
- **POST /api/v1/reprocess/{chunk_id}**: Manually trigger reprocessing for a specific chunk.
- **GET /api/v1/system/health**: Get the current health status of all components.
- **GET /api/v1/config**: Retrieve the current system configuration.
- **PUT /api/v1/config**: Update the system configuration.

### Component: Database

**Purpose:** Manages the persistence and retrieval of all system data, including chunk metadata, identity information, and system health metrics. It is not expected to be directly user-facing but provides the backbone for all other components.

**API Endpoints:**

*   This component does not expose a direct HTTP API. It interacts with other components via the MQTT message bus and direct database connections.

## MQTT Messaging Protocol

This specification defines the channels (topics) and message formats for inter-component communication via MQTT.

### Topic: `marengo/chunk_processor/chunk_processed`
- **Publisher**: ChunkProcessor
- **Subscriber**: IdentityResolver
- **Description**: Published when the ChunkProcessor has finished its processing pass on a video chunk. This signals the IdentityResolver that a new chunk is ready for identity resolution.
- **Payload**:
  ```json
  {
    "chunk_id": "string",      // Unique identifier for the chunk
    "chunk_path": "string",    // Filesystem path to the processed chunk data
    "camera_id": "string",     // Identifier for the camera that generated the chunk
    "start_time": "isodate",   // Start timestamp of the chunk
    "end_time": "isodate"      // End timestamp of the chunk
  }
  ```

### Topic: `marengo/identity_resolver/identities_resolved`
- **Publisher**: IdentityResolver
- **Subscriber**: TemporalLinker
- **Description**: Published when the IdentityResolver has finished associating identities with tracks in a chunk. This signals the TemporalLinker to begin linking tracks across chunks.
- **Payload**:
  ```json
  {
    "chunk_id": "string"       // Unique identifier for the chunk that was processed
  }
  ```

### Topic: `marengo/temporal_linker/tracks_linked`
- **Publisher**: TemporalLinker
- **Subscriber**: Dashboard, API Layer
- **Description**: Published when the TemporalLinker has completed a linking operation. This can be used to notify the UI or other services that new long-term track information is available.
- **Payload**:
  ```json
  {
    "chunk_id": "string",          // The chunk that was most recently linked
    "updated_chunk_ids": ["string"] // Array of other chunk_ids that were modified during the linking process
  }
  ```

### Topic: `marengo/system/heartbeat`
- **Publisher**: All components
- **Subscriber**: SystemHealth
- **Description**: Each component publishes a heartbeat message at a regular interval to indicate it is running.
- **Payload**:
  ```json
  {
    "component_name": "string", // e.g., "ChunkProcessor", "IdentityResolver"
    "timestamp": "isodate",
    "status": "ok" | "error",
    "message": "string"         // Optional error message
  }
  ```

### Topic: `marengo/system/health_status`
- **Publisher**: SystemHealth
- **Subscriber**: Dashboard
- **Description**: The SystemHealth component publishes a summary of the system's health status, based on the heartbeats it receives.
- **Payload**:
  ```json
  {
    "timestamp": "isodate",
    "component_statuses": [
      {
        "component_name": "string",
        "status": "ok" | "error" | "stale",
        "last_heartbeat": "isodate"
      }
    ]
  }
  ```

### Topic: `marengo/config/updated`
- **Publisher**: Configuration component
- **Subscriber**: All components
- **Description**: Published when the system configuration has been updated. Components should listen to this topic and reload their configuration.
- **Payload**:
  ```json
  {
    "timestamp": "isodate",
    "updated_keys": ["string"] // List of configuration keys that have changed
  }
  ```

### Topic: `marengo/dashboard/progress`
- **Publisher**: ChunkProcessor, IdentityResolver, TemporalLinker
- **Subscriber**: Dashboard
- **Description**: Used to send progress updates for long-running tasks to the dashboard.
- **Payload**:
  ```json
  {
    "component": "string", // "ChunkProcessor", "IdentityResolver", etc.
    "chunk_id": "string",
    "status": "processing" | "completed" | "failed",
    "progress": 0.5,       // A float between 0.0 and 1.0
    "message": "string"    // e.g., "Detecting objects"
  }
  ```

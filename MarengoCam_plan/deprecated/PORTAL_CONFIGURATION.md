# Portal Configuration

**Simple:** Portals are connections between cameras. You configure them. No learning, no statistics.

## What Portals Do

When a track ends on CamA in a portal cell, the system looks for continuation candidates on CamB in the corresponding portal cell(s).

**That's it.** Face match or LLM decides if they're the same person.

## Configuration Format

```yaml
# config/property_map.yaml
portals:
  - name: "Front Door"
    type: [person, vehicle]
    connections:
      - camera_a: "FrontPorch"
        cells_a: [[15,20], [16,20], [15,21]]  # 32×32 grid cells
        camera_b: "Entryway"
        cells_b: [[3,5], [3,6]]
        max_time: 15  # seconds (loose filter, not critical)
    
  - name: "Garage Bay"
    type: [vehicle]
    connections:
      - camera_a: "GarageInterior"
        cells_a: [[10,10], [11,10], [11,11]]
        camera_b: "Driveway"
        cells_b: [[8,12], [9,12]]
        max_time: 20
      - camera_a: "GarageInterior"
        cells_a: [[10,10], [11,10]]
        camera_b: "Backyard"
        cells_b: [[2,8]]
        max_time: 30
    
  - name: "Side Gate"
    type: [person]
    connections:
      - camera_a: "Backyard"
        cells_a: [[20,5], [20,6]]
        camera_b: "Sidewalk"
        cells_b: [[2,10]]
        max_time: 10
  
  # Exit portals (one-way, to outside world)
  - name: "Driveway Exit"
    type: [vehicle, person]
    exit: true  # Track terminates here (left property)
    camera: "Driveway"
    cells: [[30,15], [30,16], [31,15]]  # Edge of property boundary
  
  - name: "Sidewalk Exit West"
    type: [person]
    exit: true
    camera: "FrontYard"
    cells: [[28,2], [29,2], [30,2]]
  
  - name: "Sidewalk Exit East"  
    type: [person]
    exit: true
    camera: "Sidewalk"
    cells: [[25,30], [26,30]]
```

## How It Works

```python
def find_merge_candidates_via_portals(track_ended):
    """
    Track just ended. Find candidates via configured portals.
    """
    
    candidates = []
    exit_cell = track_ended.exit_cell_32x32
    exit_time = track_ended.end_time
    exit_camera = track_ended.camera
    agent_type = track_ended.agent_type
    
    # Find portals where this track exited
    portals = get_portals_for_camera_cell(exit_camera, exit_cell, agent_type)
    
    for portal in portals:
        # Check if this is an exit portal (track terminates)
        if portal.exit:
            mark_agent_left_property(track_ended.agent, portal.name, exit_time)
            return []  # No merge candidates, agent left property
        
        # Get the other side(s) of portal
        for connection in portal.connections:
            if connection.camera_a == exit_camera:
                target_camera = connection.camera_b
                target_cells = connection.cells_b
                max_time = connection.max_time
            elif connection.camera_b == exit_camera:
                target_camera = connection.camera_a
                target_cells = connection.cells_a
                max_time = connection.max_time
            else:
                continue
            
            # Find tracks starting on other side within time window
            time_window = (exit_time, exit_time + max_time)
            
            potential = query_tracks(
                camera=target_camera,
                cells_32x32=target_cells,
                time_range=time_window,
                agent_type=agent_type
            )
            
            for track in potential:
                candidates.append({
                    'source': track_ended,
                    'target': track,
                    'portal': portal.name,
                    'delta_t': track.start_time - exit_time
                })
    
    return candidates
```

## Discovery Mode (Optional, Phase 3+)

If you don't configure portals initially, system can suggest them:

```python
def suggest_portal(track_a, track_b, validated_merge):
    """
    After human/LLM confirms merge, suggest portal if none exists.
    """
    
    if not has_portal(track_a.camera, track_a.exit_cell, 
                      track_b.camera, track_b.entry_cell):
        
        suggestion = {
            'camera_a': track_a.camera,
            'cells_a': [track_a.exit_cell],  # Start with single cell
            'camera_b': track_b.camera,
            'cells_b': [track_b.entry_cell],
            'observed_time': track_b.start_time - track_a.end_time,
            'suggested_max_time': 30  # Default generous window
        }
        
        log_portal_suggestion(suggestion)
        # You review and add to config if makes sense
```

## Grid Mapping

Portals use **32×32 cells** (intra-camera grid) for precision.

When checking adjacency for neighbor camera boost, system converts to **8×8** (downsamples 4:1).

```python
def to_8x8(cell_32x32):
    row, col = cell_32x32
    return (row // 4, col // 4)

# Example: Portal at 32×32 (15,20) maps to 8×8 (3,5)
```

## Why No Statistics?

Original design had:
- `observations[]` - list of observed travel times
- `min_time`, `typical_time`, `median()` calculations
- Support counts and validation tracking

**But:** This only helps narrow the candidate pool. If you have:
- 0 candidates → track ends (no portal or person went elsewhere)
- 1 candidate → probably same person (check with face/LLM)
- 3 candidates → ask LLM which one
- 10 candidates → mark uncertain

Knowing "typical time is 3.2s vs 8.5s" doesn't change the decision logic. You're asking LLM either way.

**Keep it simple:** Portal = adjacency filter. Time = loose sanity check (< 60s). Done.

## Exit Portals (Track Termination)

When track ends at an exit portal, agent is marked as "left property":

```python
def mark_agent_left_property(agent, exit_portal, timestamp):
    """
    Agent exited property boundary. Stop tracking.
    """
    agent.current_state = {
        'visible': False,
        'left_property': True,
        'exit_portal': exit_portal,
        'exit_time': timestamp
    }
    
    # Useful for vehicle associations:
    # "Person walked to Driveway Exit at 14:32, vehicle departed at 14:33"
    # → Person probably in vehicle
```

**Use cases:**
- Person walks to sidewalk exit → track ends, they left
- Person walks to driveway exit, vehicle departs 10s later → associate with vehicle
- Vehicle exits driveway → terminates on property, might return later

## Schema

```python
# Configuration (YAML file, user edits)
Portal:
    name: str
    type: list[str]  # ['person', 'vehicle']
    exit: bool = False  # If true, track terminates (left property)
    
    # For regular portals (exit=false):
    connections: list[Connection]
    
    # For exit portals (exit=true):
    camera: str
    cells: list[tuple[int, int]]  # 32×32 cells

Connection:
    camera_a: str
    cells_a: list[tuple[int, int]]  # 32×32 cells
    camera_b: str
    cells_b: list[tuple[int, int]]
    max_time: int  # seconds (loose filter, default 60)

# Runtime (in-memory only, no persistence)
# System just loads config and uses it for candidate queries
```

## Configuration UI (Phase 3)

Simple web tool:
1. Show camera thumbnail with 32×32 grid overlay
2. Click cells to mark portal zones
3. Link to other camera's portal zones
4. Set name, type, max_time
5. Save to `property_map.yaml`

No visualization of "learned edges" or "confidence growth" - there's nothing to visualize.

## Phase Plan

**Phase 2:** No portals, wide discovery window (any camera, Δt < 60s, use face/LLM)

**Phase 3:** Add portal configuration
- Manually mark your known doors/gates
- System only checks configured portals for candidates
- Reduces LLM calls dramatically (fewer false candidates)

**Phase 4+:** Optional discovery suggestions
- After validated merges, system logs "no portal exists for this path"
- You review suggestions and add to config if useful

**No statistics, no learning, no complexity.**

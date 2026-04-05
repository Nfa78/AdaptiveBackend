# Data Definition: Adaptive VR Rehab System

This document defines the core data model and the performance features used by the Adaptive VR Rehabilitation System. It separates **global performance metrics** (computed for every task in every scene) from **scene-specific metrics** (computed only when relevant).

## Design Principles
- Global metrics are **stable, comparable, and required** for all tasks.
- Scene-specific metrics are **optional, extensible, and diagnostic**.
- Adaptive decisions in Phase 1 use **global metrics only**.
- Scene-specific metrics are stored in a flexible structure to avoid schema churn.

## Tables

### 1) `patients`
Stores anonymized participant identity.
- `id` (UUID, PK)
- `anon_code` (text, unique)
- `age_range` (text, optional)
- `dominant_hand` (text, optional)
- `notes` (text, optional)

### 2) `sessions`
Represents a therapy session (one visit).
- `id` (UUID, PK)
- `patient_id` (UUID, FK -> patients.id)
- `start_time` (timestamp)
- `end_time` (timestamp)
- `device` (text) — headset/controller info
- `version` (text) — app/build version
- `notes` (text, optional)

### 3) `tasks`
One executed task within a session.
- `id` (UUID, PK)
- `session_id` (UUID, FK -> sessions.id)
- `scene` (text) — e.g., `kitchen`, `living_room`, `garden`
- `task_type` (text) — e.g., `watering`, `picking`, `hanging`, `cleaning`
- `difficulty_level` (int)
- `target_size` (float) — meters or normalized
- `reach_distance` (float) — meters
- `task_complexity` (int) — number of steps/checkpoints
- `start_time` (timestamp)
- `end_time` (timestamp)

### 4) `performance_metrics_global`
Global metrics computed for **every task**.
- `task_id` (UUID, PK, FK -> tasks.id)
- `completion_time_norm` (float)
- `error_count` (int)
- `prompt_count` (int)
- `path_efficiency` (float)
- `smoothness` (float)
- `hesitation_count` (int)
- `hesitation_total_time` (float)
- `spatial_accuracy` (float)
- `avg_speed` (float, optional)
- `peak_speed` (float, optional)

### 5) `performance_metrics_scene`
Scene- or task-specific metrics stored as key–value pairs.
- `id` (UUID, PK)
- `task_id` (UUID, FK -> tasks.id)
- `feature_name` (text)
- `feature_value` (float)
- `feature_unit` (text, optional)

### 6) `adaptation_decisions`
Records difficulty adjustments made by the adaptive system.
- `id` (UUID, PK)
- `task_id` (UUID, FK -> tasks.id)
- `method` (text) — `rule_based` or `ml`
- `decision` (text) — `increase`, `maintain`, `decrease`
- `confidence` (float, optional)
- `timestamp` (timestamp)

## Global Performance Metrics (Definitions)

### 1) Normalized Completion Time
**Definition:** time to complete task normalized by difficulty or expected time.
**Example:** `completion_time_norm = actual_time / expected_time(difficulty)`
**Why it matters:** captures performance independent of task difficulty.

### 2) Error Count
**Definition:** number of task rule violations (drops, misplacements, spills, misses).
**Why it matters:** reflects control and accuracy.

### 3) Prompt Count
**Definition:** number of guidance requests or prompts triggered.
**Why it matters:** proxies cognitive load and task confidence.

### 4) Path Efficiency
**Definition:** `straight_line_distance / actual_path_length`.
**Why it matters:** measures movement economy and directness.

### 5) Movement Smoothness
**Definition:** variability in velocity/acceleration or jerk-based proxy.
**Why it matters:** smoother motion indicates improved motor control.

### 6) Hesitation
**Definition:** pauses during task execution where hand/controller speed stays below a low threshold for at least a minimum duration.
- hesitation_count = number of pauses
- hesitation_total_time = total paused time (seconds)

**Detection:** smooth positions, compute speed (t), and mark a pause when (t) < v_thresh continuously for 	_min (e.g., 0.5–1.0 s). Count and sum durations only between 	ask_start and 	ask_end, excluding menus, teleport transitions, or cutscenes.

**Threshold selection:** use a patient/session baseline (e.g., _thresh = 0.2 * median_speed) or a task-relative percentile to avoid over-counting in low-mobility users.

**Why it matters:** indicates uncertainty, fatigue, or planning difficulty.

### 7) Spatial Accuracy
**Definition:** mean distance between actual trajectory and ideal path.
**Why it matters:** measures precision in reaching/manipulation.

### 8) Speed Metrics (optional)
- `avg_speed`, `peak_speed` for additional motion profiling.

## Scene-Specific Metrics (Examples)
Store these in `performance_metrics_scene` as `{feature_name, feature_value}`.

### Kitchen Scene
- `pour_spill_volume` (ml)
- `fill_accuracy` (%)
- `cut_precision` (mm)

### Living Room Scene
- `snap_alignment_error` (mm)
- `object_orientation_error` (degrees)

### Garden Scene
- `watering_coverage` (%)
- `tool_use_time` (seconds)

## Modeling Notes
- **Global metrics** are mandatory and computed for every task.
- **Scene-specific metrics** are optional and only logged when relevant.
- Keep global metrics consistent across tasks to ensure comparability.
- Scene-specific metrics allow detailed analysis without changing schema.

## Example API Payload
```json
{
  "task_id": "uuid",
  "scene": "kitchen",
  "task_type": "watering",
  "difficulty_level": 2,
  "global_metrics": {
    "completion_time_norm": 1.12,
    "error_count": 1,
    "prompt_count": 0,
    "path_efficiency": 0.86,
    "smoothness": 0.72,
    "hesitation_count": 2,
    "hesitation_total_time": 3.4,
    "spatial_accuracy": 0.04
  },
  "scene_metrics": [
    {"feature_name": "pour_spill_volume", "feature_value": 12.0, "feature_unit": "ml"},
    {"feature_name": "fill_accuracy", "feature_value": 0.92, "feature_unit": "ratio"}
  ]
}
```





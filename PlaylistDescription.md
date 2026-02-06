# InkyPi Playlist System Overview

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Core Components](#core-components)
3. [How Playlists Work](#how-playlists-work)
4. [Plugin Instance Lifecycle](#plugin-instance-lifecycle)
5. [Refresh Mechanisms](#refresh-mechanisms)
6. [Data Flow](#data-flow)
7. [Time-Based Scheduling](#time-based-scheduling)
8. [Configuration Management](#configuration-management)

---

## System Architecture

The playlist system is built around a background thread that continuously manages display updates based on time-based playlists and plugin-specific refresh intervals.

### Key Files
- [src/refresh_task.py](src/refresh_task.py) - Background refresh logic
- [src/model.py](src/model.py) - Data models (Playlist, PluginInstance, PlaylistManager, RefreshInfo)
- [src/config.py](src/config.py) - Configuration management
- [src/blueprints/playlist.py](src/blueprints/playlist.py) - Playlist CRUD API endpoints
- [src/blueprints/plugin.py](src/blueprints/plugin.py) - Plugin instance management and manual updates

---

## Core Components

### 1. RefreshTask (Background Thread)
**Location**: [src/refresh_task.py](src/refresh_task.py#L15)

The `RefreshTask` class runs as a daemon thread and manages all display updates.

**Main responsibilities:**
- Runs in an infinite loop with configurable sleep intervals
- Determines which plugin to display next based on active playlists
- Handles both automatic (playlist-based) and manual updates
- Compares image hashes to avoid unnecessary display refreshes
- Updates refresh metadata after each display update

**Key method**: `_run()` - The main background loop that executes every `plugin_cycle_interval_seconds` (default: 3600 seconds / 1 hour)

### 2. PlaylistManager
**Location**: [src/model.py](src/model.py#L63)

Manages multiple time-based playlists.

**Key features:**
- Stores a list of `Playlist` objects
- Tracks the currently `active_playlist`
- Determines which playlist is active based on current time
- Handles playlist CRUD operations

### 3. Playlist
**Location**: [src/model.py](src/model.py#L177)

Represents a time-bounded collection of plugin instances.

**Attributes:**
- `name` - Unique playlist identifier
- `start_time` - When playlist becomes active (format: "HH:MM")
- `end_time` - When playlist becomes inactive (format: "HH:MM", can be "24:00")
- `plugins` - List of `PluginInstance` objects
- `current_plugin_index` - Tracks which plugin to show next

**Important methods:**
- `is_active(current_time)` - Checks if current time falls within start/end range (handles midnight wrap-around)
- `get_next_plugin()` - Returns the next plugin and advances the index (round-robin rotation)
- `get_priority()` - Returns time range in minutes (smaller range = higher priority)

### 4. PluginInstance
**Location**: [src/model.py](src/model.py#L285)

Represents a single instance of a plugin within a playlist.

**Attributes:**
- `plugin_id` - Reference to the plugin type (e.g., "clock", "weather")
- `name` - User-defined instance name (e.g., "Living Room Clock")
- `settings` - Plugin-specific configuration
- `refresh` - Refresh schedule configuration (interval or scheduled)
- `latest_refresh_time` - ISO timestamp of last refresh

**Refresh configuration formats:**
```python
# Interval-based (every X seconds)
{"interval": 3600}  # Refresh every hour

# Scheduled (specific time of day)
{"scheduled": "14:30"}  # Refresh at 2:30 PM daily
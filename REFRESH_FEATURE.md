# Refresh Feature Implementation

This document describes the newly implemented refresh functionality in Serie.

## Features Implemented

### 1. Manual Refresh (Keyboard Input)
Users can now manually refresh the repository data by pressing:
- **`r`** key
- **`F5`** key

When triggered, Serie will:
- Reload all git commit data
- Recalculate the commit graph
- Regenerate graph images
- Update the display with the latest repository state

### 2. Auto-Refresh (Optional)
Serie can now automatically refresh repository data at regular intervals.

#### Configuration
Add the following to your `~/.config/serie/config.toml`:

```toml
[core.option]
auto_refresh = true                    # Enable auto-refresh (default: false)
auto_refresh_interval_secs = 5         # Refresh every 5 seconds (default: 5)
```

**Note:** Auto-refresh is disabled by default to maintain backward compatibility.

## Technical Details

### Architecture Changes

1. **Event System** (`src/event.rs`):
   - Added `UserEvent::Refresh` for keyboard-triggered refresh
   - Added `AppEvent::Refresh` for internal refresh events
   - Modified `event::init()` to accept auto-refresh parameters
   - Added optional timer thread that sends `AppEvent::Refresh` at intervals
   - Made `Receiver` cloneable using `Arc<Mutex<>>` to support refresh loop

2. **Keybindings** (`assets/default-keybind.toml`):
   - Added `refresh = ["r", "f5"]` mapping

3. **Configuration** (`src/config.rs`):
   - Added `auto_refresh: bool` to `CoreOptionConfig`
   - Added `auto_refresh_interval_secs: u64` to `CoreOptionConfig`

4. **Application** (`src/app.rs`):
   - Added `AppResult` enum to distinguish between Quit and Refresh
   - Modified `App::run()` to return `AppResult` instead of `()`
   - Added handler for `UserEvent::Refresh` that sends `AppEvent::Refresh`
   - Made `InitialSelection` derive `Copy` to support refresh loop

5. **Main Loop** (`src/lib.rs`):
   - Restructured to support refresh loop
   - Repository, graph, and app are now recreated on each refresh
   - Terminal instance is preserved across refreshes

### Refresh Flow

```
User presses 'r' or F5
    ↓
KeyEvent captured by input thread
    ↓
Mapped to UserEvent::Refresh
    ↓
Converted to AppEvent::Refresh
    ↓
App::run() returns AppResult::Refresh
    ↓
Main loop continues (doesn't break)
    ↓
Repository::load() called again
    ↓
Graph recalculated
    ↓
New App instance created
    ↓
User sees updated data
```

### Auto-Refresh Flow

```
Timer thread sleeping
    ↓
Interval elapses
    ↓
Timer sends AppEvent::Refresh
    ↓
Same flow as manual refresh
```

## Usage Examples

### Manual Refresh Only (Default)
No configuration needed. Just press `r` or `F5` in the application.

### Auto-Refresh Every 10 Seconds
```toml
[core.option]
auto_refresh = true
auto_refresh_interval_secs = 10
```

### Auto-Refresh Every 30 Seconds
```toml
[core.option]
auto_refresh = true
auto_refresh_interval_secs = 30
```

## Use Cases

1. **Monitoring Active Development**: Watch commits appear in real-time as you or your team commits
2. **CI/CD Monitoring**: Keep track of automated commits from CI systems
3. **Code Review**: Refresh to see new commits without restarting the application
4. **Long-Running Sessions**: Keep the view up-to-date during extended use

## Performance Considerations

- Refresh involves running git commands and recalculating the graph
- For large repositories, this may take a moment
- Auto-refresh interval should be adjusted based on repository size
- Recommended minimum interval: 5 seconds for large repos, 2-3 seconds for smaller repos

## Backward Compatibility

All changes are backward compatible:
- Auto-refresh is **disabled by default**
- Manual refresh keybindings don't conflict with existing bindings
- Configuration is optional
- Existing installations work without modification

## Testing

The implementation has been tested with:
- Build verification: ✅ Compiles successfully
- Default behavior: Manual refresh via keyboard
- Optional auto-refresh with configurable intervals

## Future Enhancements (Suggestions)

1. Add visual feedback when refresh is in progress
2. Show timestamp of last refresh
3. Detect filesystem changes instead of polling
4. Allow disabling manual refresh keybindings
5. Add refresh shortcut to help menu

# Compaction Corruption Error Handling

## Overview

This feature prevents infinite retry loops when tablets encounter persistent corruption errors during compaction operations. When a tablet has corrupted data (e.g., "Bad page: too small size (0)" errors), the system now implements exponential backoff to avoid wasting resources retrying the same failed operation.

## Problem Solved

Prior to this change:
- Tablets with corrupted data would be continuously selected for compaction
- Each compaction attempt would fail with the same corruption error
- The tablet would immediately be re-queued for compaction
- This created an infinite loop, wasting CPU and I/O resources

## Solution

The system now tracks consecutive corruption errors per tablet and implements exponential backoff:

1. **Corruption Detection**: When a compaction fails with `Status::is_corruption()`, the error is tracked
2. **Consecutive Error Tracking**: Each consecutive corruption error increments a counter
3. **Exponential Backoff**: After each corruption error, the wait time doubles (up to a maximum)
4. **Automatic Reset**: Successful compaction or non-corruption errors reset the counter

## Configuration Parameters

Three new configuration parameters control this behavior:

### `max_consecutive_compaction_corruption_errors`
- **Default**: 3
- **Type**: int32
- **Description**: Maximum consecutive corruption errors before backoff is applied
- **Special Values**: -1 disables this protection (not recommended)

### `compaction_corruption_backoff_base_seconds`
- **Default**: 600 (10 minutes)
- **Type**: int32
- **Description**: Base time for exponential backoff calculation
- **Formula**: `actual_backoff = base * 2^(error_count - 1)`

### `compaction_corruption_backoff_max_seconds`
- **Default**: 86400 (24 hours)
- **Type**: int32
- **Description**: Maximum backoff time to prevent indefinitely long waits

## Backoff Schedule Example

With default settings (base=600s, max=86400s):

| Consecutive Errors | Backoff Time |
|-------------------|--------------|
| 1                 | 10 minutes   |
| 2                 | 20 minutes   |
| 3                 | 40 minutes   |
| 4                 | 80 minutes   |
| 5                 | 160 minutes  |
| 6                 | 320 minutes  |
| 7+                | 24 hours (max)|

## Monitoring

### Log Messages

When corruption is detected:
```
W[timestamp] Compaction corruption error detected for tablet [tablet_id], 
consecutive errors: [count], backoff time: [seconds] seconds, 
error: [error_details]
```

When max consecutive errors is reached:
```
W[timestamp] Tablet [tablet_id] has reached maximum consecutive corruption errors ([count]). 
This tablet may have persistent data corruption and will use exponential backoff for compaction retries.
```

During backoff period (VLOG level 2):
```
Tablet [tablet_id] is in corruption backoff period. 
Error count: [count], backoff time: [seconds] seconds, time remaining: [seconds] seconds
```

### JSON Status API

The tablet compaction status endpoint now includes corruption tracking information:

```json
{
  "consecutive_corruption_errors": 3,
  "last_corruption_time": "2025-10-16 20:08:21",
  "corruption_backoff_remaining_seconds": 1234
}
```

Access via: `/api/compaction/show?tablet_id=[id]`

## Implementation Details

### Modified Files

1. **be/src/common/config.h**
   - Added three new configuration parameters

2. **be/src/storage/tablet.h**
   - Added `_consecutive_compaction_corruption_errors` atomic counter
   - Added `_last_compaction_corruption_millis` timestamp tracking
   - Added helper methods for corruption tracking

3. **be/src/storage/tablet.cpp**
   - Modified `need_compaction()` to check backoff status
   - Modified `force_base_compaction()` to check backoff status
   - Enhanced `get_compaction_status()` to include corruption info

4. **be/src/storage/compaction_task.cpp**
   - Modified `_success_callback()` to reset corruption counter
   - Modified `_failure_callback()` to detect and track corruption errors

### Thread Safety

- All corruption tracking uses atomic operations (`std::atomic<>`)
- No new locks are introduced
- Backoff checks occur within existing `_compaction_task_lock`

## Operational Considerations

### When Corruption is Detected

1. **Investigation**: Check BE logs for the specific corruption error and affected file
2. **Data Recovery**: Consider:
   - Restoring from backup
   - Dropping and reloading the affected partition
   - Manual cleanup of corrupted rowset files (advanced users only)
3. **Monitoring**: Watch the corruption backoff metrics to identify tablets with persistent issues

### Tuning Parameters

- **Increase `max_consecutive_compaction_corruption_errors`**: For environments with transient corruption that may resolve
- **Increase `compaction_corruption_backoff_base_seconds`**: To reduce retry frequency in heavily loaded systems
- **Decrease `compaction_corruption_backoff_max_seconds`**: For faster retry in systems with good monitoring and alerting

### Disabling the Feature

Set `max_consecutive_compaction_corruption_errors = -1` to disable corruption-specific backoff. This is **not recommended** as it may lead to resource waste, but can be useful for debugging.

## Future Enhancements

Potential improvements for future versions:

1. **Metrics Export**: Add StarRocks metrics for tablets with corruption
2. **Admin API**: Provide API to manually reset corruption counters
3. **Alerting**: Integrate with monitoring systems for corruption alerts
4. **Auto-remediation**: Automatically mark severely corrupted tablets as offline

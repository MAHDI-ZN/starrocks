# Solution Summary: Fixing Infinite Compaction Loop on Data Corruption

## Original Problem (from logs)

The user reported seeing these repeating errors in their backend logs:

```
"Oct 16, 2025 @ 20:08:21.809","W20251016 20:08:21.809290 140560954246720 compaction_task.cpp:140] compaction finish. status:Internal error: reader get_next error: Corruption: Bad page: too small size (0), file(/opt/starrocks/be/storage/data/136/13208/395737339/0200000000007d9772400ed03c9e4dc25ba8cf0a413629a9_0.dat)","starrocks-sharednothing-be-5"
```

**The Problem:**
- Tablet 13208 had corrupted data (bad page with size 0)
- Compaction was continuously attempted every few minutes
- Each attempt failed with the same error
- This created an infinite loop, wasting resources

## Root Cause Analysis

1. **Corruption Detection**: The file `be/src/storage/rowset/page_io.cpp` properly detects corrupted pages:
   ```cpp
   if (page_size < 8) {
       return Status::Corruption(
           strings::Substitute("Bad page: too small size ($0), file($1)", 
                             page_size, opts.read_file->filename()));
   }
   ```

2. **Missing Retry Logic**: When compaction failed with corruption:
   - `compaction_task.cpp` would call `_failure_callback()` 
   - The failure was logged but no special handling for corruption
   - Tablet was immediately re-queued for next compaction round
   - No backoff or delay mechanism existed

3. **Selection Logic**: In `tablet.cpp`:
   - `need_compaction()` had no awareness of previous corruption errors
   - Would always return true if compaction score was high
   - No mechanism to temporarily exclude corrupted tablets

## Solution Implementation

### 1. Configuration (config.h)
Added three tunable parameters to control behavior:

```cpp
CONF_mInt32(max_consecutive_compaction_corruption_errors, "3");
CONF_mInt32(compaction_corruption_backoff_base_seconds, "600");
CONF_mInt32(compaction_corruption_backoff_max_seconds, "86400");
```

### 2. State Tracking (tablet.h)
Added atomic variables to track corruption state per tablet:

```cpp
std::atomic<int32_t> _consecutive_compaction_corruption_errors{0};
std::atomic<int64_t> _last_compaction_corruption_millis{0};
```

### 3. Error Detection (compaction_task.cpp)
Enhanced failure callback to detect and track corruption:

```cpp
void CompactionTask::_failure_callback(const Status& st) {
    if (st.is_corruption()) {
        _tablet->increment_compaction_corruption_errors();
        _tablet->set_last_compaction_corruption_time(UnixMillis());
        // Calculate exponential backoff...
        LOG(WARNING) << "Compaction corruption error detected..."
    } else {
        _tablet->reset_compaction_corruption_errors();
    }
}
```

### 4. Backoff Logic (tablet.cpp)
Modified compaction selection to apply exponential backoff:

```cpp
bool Tablet::need_compaction() {
    // Check if in backoff period
    int32_t error_count = _consecutive_compaction_corruption_errors.load();
    if (error_count > 0) {
        int64_t backoff_ms = calculate_exponential_backoff(error_count);
        if (still_in_backoff_period(backoff_ms)) {
            return false;  // Skip this tablet
        }
    }
    // Normal compaction logic...
}
```

### 5. Observability
Added JSON status endpoint information:

```json
{
  "consecutive_corruption_errors": 3,
  "last_corruption_time": "2025-10-16 20:08:21",
  "corruption_backoff_remaining_seconds": 1200
}
```

## How It Fixes The Original Problem

**Before this change:**
```
Time 20:08:21 - Corruption error → Log warning
Time 20:08:27 - Picked again → Corruption error → Log warning
Time 20:10:26 - Picked again → Corruption error → Log warning
Time 20:12:35 - Picked again → Corruption error → Log warning
... (infinite loop)
```

**After this change:**
```
Time 20:08:21 - Corruption error #1 → Log warning + set 10-min backoff
Time 20:08:27 - Tablet checked → Skipped (in backoff)
Time 20:10:26 - Tablet checked → Skipped (in backoff)
Time 20:12:35 - Tablet checked → Skipped (in backoff)
Time 20:18:21 - Backoff expired → Retry compaction
  If success → Counter reset to 0
  If corruption error #2 → Set 20-min backoff
```

## Backoff Schedule (Default Config)

| Error # | Backoff Time | Next Retry After |
|---------|--------------|------------------|
| 1       | 10 minutes   | First error + 10m |
| 2       | 20 minutes   | Second error + 20m |
| 3       | 40 minutes   | Third error + 40m |
| 4       | 80 minutes   | Fourth error + 80m |
| 5       | 160 minutes  | Fifth error + 160m |
| 6+      | 24 hours     | Max backoff reached |

## Key Benefits

1. **Resource Efficiency**: Stops wasting CPU/IO on repeatedly failing compactions
2. **Log Clarity**: Reduces log spam from repeated failures
3. **Operational Visibility**: Admins can see which tablets have corruption via API
4. **Configurability**: Can be tuned or disabled based on operational needs
5. **Self-Healing**: Automatically resets on successful compaction
6. **Thread Safety**: Uses atomic operations, no new locks

## Example Log Output

When corruption is detected:
```
W[timestamp] Compaction corruption error detected for tablet 13208, 
consecutive errors: 1, backoff time: 600 seconds, 
error: Internal error: reader get_next error: Corruption: Bad page: too small size (0)
```

When max errors reached:
```
W[timestamp] Tablet 13208 has reached maximum consecutive corruption errors (3). 
This tablet may have persistent data corruption and will use exponential backoff 
for compaction retries.
```

During backoff (VLOG level 2):
```
Tablet 13208 is in corruption backoff period. Error count: 3, 
backoff time: 2400 seconds, time remaining: 1800 seconds
```

## Operational Workflow

When corruption is detected, operators can:

1. **Check tablet status**:
   ```
   curl http://be_host:be_http_port/api/compaction/show?tablet_id=13208
   ```

2. **Investigate the error**:
   - Check BE logs for the specific file causing corruption
   - Verify disk health
   - Check for recent system issues

3. **Remediate**:
   - Restore from backup if available
   - Drop and reload the affected partition
   - Wait for natural recovery (if transient)

4. **Monitor**:
   - Watch `consecutive_corruption_errors` metric
   - Set alerts for tablets with high error counts

## Configuration Tuning

**For production with good monitoring:**
```
max_consecutive_compaction_corruption_errors=3
compaction_corruption_backoff_base_seconds=600  # 10 minutes
compaction_corruption_backoff_max_seconds=86400 # 24 hours
```

**For aggressive retry (development/testing):**
```
max_consecutive_compaction_corruption_errors=5
compaction_corruption_backoff_base_seconds=300  # 5 minutes
compaction_corruption_backoff_max_seconds=7200  # 2 hours
```

**For conservative approach (heavily loaded systems):**
```
max_consecutive_compaction_corruption_errors=2
compaction_corruption_backoff_base_seconds=1800 # 30 minutes
compaction_corruption_backoff_max_seconds=86400 # 24 hours
```

**To disable (not recommended):**
```
max_consecutive_compaction_corruption_errors=-1
```

## Conclusion

This solution:
- ✅ Stops the infinite retry loop described in the original issue
- ✅ Preserves system resources by avoiding futile retry attempts
- ✅ Provides visibility into which tablets have corruption issues
- ✅ Allows natural recovery if corruption is transient
- ✅ Gives operators time to investigate and remediate
- ✅ Is configurable to suit different operational requirements
- ✅ Maintains thread safety and performance

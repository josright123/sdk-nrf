# DM9051 Stack Overflow Fix - Configuration Changes

## Problem Analysis

The error log showed:
```
ASSERTION FAIL [net_buf_simple_tailroom(buf) >= len] @ WEST_TOPDIR/zephyr/lib/net_buf/buf_simple.c:62
***** USAGE FAULT *****
  Stack overflow (context area not valid)
>>> ZEPHYR FATAL ERROR 2: Stack overflow on CPU 0
Current thread: 0x200003c8 (dm9051_rx)
```

### Root Cause
The `dm9051_rx` thread was running out of stack space. The default stack size (typically 1024-1536 bytes) was insufficient for the packet processing operations, particularly when handling network buffers and calling Zephyr's networking stack functions.

## Solution Applied

### Configuration Changes in `prj.conf`

Added two critical configuration options:

1. **CONFIG_ETH_DM9051_RX_THREAD_STACK_SIZE=4096**
   - Increases the RX thread stack from default (~1024-1536 bytes) to 4096 bytes
   - Provides sufficient space for packet processing, buffer operations, and function call overhead
   - This is the primary fix for the stack overflow issue

2. **CONFIG_ETH_DM9051_TIMEOUT=1000**
   - Sets timeout to 1000 milliseconds (1 second)
   - Ensures reasonable timeout for network operations

## Why This Fixes the Issue

The stack overflow occurred because:
1. The `dm9051_rx` thread performs complex operations including:
   - SPI communication with the DM9051 chip
   - Network buffer allocation (`net_pkt_rx_alloc_with_buffer`)
   - Memory operations (`dm9051_read_mem`)
   - Buffer manipulation (`net_buf_add`)
   - Logging and error handling

2. Each of these operations requires stack space for:
   - Local variables
   - Function call frames
   - Return addresses
   - Register preservation

3. The default stack size was too small to accommodate all these operations, especially when combined with interrupt handling and nested function calls.

## Additional Recommendations

### 1. Add RX Length Validation (Recommended)

Add this validation in the RX packet handling code to prevent buffer overflow:

```c
/* Validate rx_len before allocation */
if (rx_len > NET_ETH_MAX_FRAME_SIZE || rx_len < NET_ETH_MIN_FRAME_SIZE) {
    LOG_ERR("%s: Invalid RX length: %d", dev->name, rx_len);
    eth_stats_update_errors_rx(context->iface);
    return -EINVAL;
}
```

This prevents attempting to allocate buffers for corrupted or invalid packet lengths.

### 2. Enable Stack Overflow Detection (Already Enabled)

Your configuration already has:
- `CONFIG_ASSERT=y` - Assertions are enabled
- This helps catch stack overflow issues early

### 3. Monitor Stack Usage

After rebuilding, you can monitor actual stack usage by enabling:
```
CONFIG_THREAD_STACK_INFO=y
CONFIG_THREAD_MONITOR=y
```

Then use the shell command `kernel stacks` to see actual stack usage.

## Next Steps

1. **Clean rebuild** the project to ensure the new configuration is applied:
   ```bash
   west build -b nrf54l15dk_nrf54l15_cpuapp -p
   ```

2. **Flash and test** the updated firmware

3. **Monitor the logs** to verify:
   - No more stack overflow errors
   - No assertion failures
   - Successful packet reception

4. **If issues persist**, consider:
   - Increasing stack size further (try 8192)
   - Adding the RX length validation mentioned above
   - Checking for memory corruption in the SPI communication

## Configuration Summary

The complete DM9051 configuration in `prj.conf` now includes:

```conf
# Ethernet L2 layer support (required for DM9051)
CONFIG_NET_L2_ETHERNET=y

# DM9051 Ethernet controller driver
CONFIG_ETH_DM9051=y

# DM9051 RX Thread Stack Size - Increased to prevent stack overflow
CONFIG_ETH_DM9051_RX_THREAD_STACK_SIZE=4096

# DM9051 Timeout configuration (in milliseconds)
CONFIG_ETH_DM9051_TIMEOUT=1000

# SPI support (DM9051 uses SPI interface)
CONFIG_SPI=y
```

## Expected Outcome

With these changes, the `dm9051_rx` thread should have sufficient stack space to:
- Handle packet reception without overflow
- Process network buffers correctly
- Execute all necessary networking stack operations
- Avoid the assertion failure and kernel panic

The error should no longer occur, and the DM9051 driver should operate reliably.

# mdadm Discord Webhook Notifications

A single bash script that sends mdadm (Linux Software RAID) events to Discord as formatted webhook notifications. This script is designed to work with mdadm's `PROGRAM` configuration option to provide real-time RAID monitoring notifications.

**Important:** Requires mdadm monitoring to be active (typically via the `mdmonitor` service) for the `PROGRAM` directive to work.

No installation scripts or dependencies required - just download, configure, and use!

## Features

- 🚨 **Real-time notifications** for all mdadm events (failures, rebuilds, etc.)
- 🎨 **Color-coded Discord embeds** based on event severity
- 🖥️ **System information** including array status, RAID level, and uptime
- 🔧 **Easy installation** with single curl command
- 📝 **Comprehensive event handling** for all mdadm event types
- 🛡️ **Error handling** with retries and proper logging

## Quick Start

### Prerequisites

- Linux system with mdadm installed
- `curl` command available
- Discord webhook URL (see [Creating a Discord Webhook](#creating-a-discord-webhook))
- Root access for installation

### Installation

**One-liner Installation** :
```bash
sudo curl -fsSL https://raw.githubusercontent.com/tdharris/os-mdadm-alert-webhook/main/mdadm-discord-webhook.sh -o /usr/bin/mdadm-discord-webhook && sudo chmod +x /usr/bin/mdadm-discord-webhook
```

Test the installation:
```bash
mdadm-discord-webhook --help
```

### Configuration

1. **Get your Discord webhook URL** (see instructions below)

2. **Configure the script**:
   Edit the webhook URL in the installed script:
   ```bash
   sudo vi /usr/bin/mdadm-discord-webhook
   ```
   Find and update this line at the top of the script:
   ```bash
   DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/YOUR_WEBHOOK_ID/YOUR_WEBHOOK_TOKEN"
   ```
   
   **Note:** Bot name and avatar are configured in Discord when you create the webhook.

3. **Configure mdadm.conf**:
   Edit `/etc/mdadm.conf` or `/etc/mdadm/mdadm.conf` and add:
   ```
   PROGRAM /usr/bin/mdadm-discord-webhook
   ```

4. **Test the configuration**:
   ```bash
   sudo mdadm-discord-webhook "TestMessage" "/dev/md0"
   ```

5. **Enable mdadm monitoring**:
   The `PROGRAM` directive only works when mdadm is actively monitoring arrays. 
   
    ```bash
    # Check that monitoring is active
    sudo systemctl status mdmonitor
    ```

    Enable monitoring:
    ```bash
    # Enable and start mdadm monitoring service
    sudo systemctl enable mdmonitor
    sudo systemctl start mdmonitor

    # Check that monitoring is active
    sudo systemctl status mdmonitor
    ```

## Usage

### Command Line Usage

```bash
mdadm-discord-webhook <event> <device> [component]
```

**Arguments:**
- `event`: mdadm event type (Fail, TestMessage, RebuildStarted, etc.)
- `device`: MD device name (e.g., /dev/md0)
- `component`: Optional component device (e.g., /dev/sda1)

**Note:** The Discord webhook URL is configured directly in the script file.

### Examples

```bash
# Test message
sudo mdadm-discord-webhook "TestMessage" "/dev/md0"

# Simulate device failure
sudo mdadm-discord-webhook "Fail" "/dev/md0" "/dev/sda1"

# Rebuild notification
sudo mdadm-discord-webhook "RebuildStarted" "/dev/md0"
```

## Supported Events

The script handles all standard mdadm events with appropriate color coding:

| Event | Description | Color |
|-------|-------------|--------|
| `Fail` | Device failure | 🔴 Red |
| `FailSpare` | Spare device failure | 🔴 Red |
| `DegradedArray` | Array degraded | 🔴 Red |
| `SparesMissing` | Missing spare devices | 🔴 Red |
| `RebuildStarted` | Rebuild started | 🟠 Orange |
| `RebuildNN` | Rebuild progress | 🟠 Orange |
| `RebuildFinished` | Rebuild completed | 🟠 Orange |
| `TestMessage` | Test notification | 🟢 Green |
| `NewArray` | New array created | 🟢 Green |
| `SpareActive` | Spare activated | 🟢 Green |
| `DeviceDisappeared` | Device disappeared | 🟠 Orange |
| `MoveSpare` | Spare moved | 🟠 Orange |

## Configuration Files

### Example mdadm.conf

Here's a complete example of `/etc/mdadm.conf`:

```bash
# mdadm.conf
#
# Please refer to mdadm.conf(5) for information about this file.
#

# ...

# Discord webhook notifications
PROGRAM /usr/bin/mdadm-discord-webhook
```

### Understanding mdadm Monitoring

The `PROGRAM` directive in mdadm.conf only works when mdadm is running in monitor mode. This typically happens through:

- **Systemd service** (recommended): The `mdmonitor` service runs `mdadm --monitor --scan` automatically
- **Manual monitoring**: Running `mdadm --monitor --scan` directly
- **Custom scripts**: Using mdadm monitoring in your own automation

**Enable automatic monitoring:**
```bash
# Enable and start mdadm monitoring service
sudo systemctl enable mdmonitor
sudo systemctl start mdmonitor

# Verify monitoring is active
sudo systemctl status mdmonitor

# Check what arrays are being monitored
sudo mdadm --detail --scan
```

**Manual monitoring (for testing):**
```bash
# Monitor all arrays defined in mdadm.conf
sudo mdadm --monitor --scan --verbose

# Monitor specific array
sudo mdadm --monitor /dev/md0 --verbose
```

## Advanced Configuration

### Environment Variables

You can set these environment variables to customize behavior:

```bash
# Custom timeout for curl requests (default: 30 seconds)
export MDADM_WEBHOOK_TIMEOUT=60

# Custom retry count (default: 3)
export MDADM_WEBHOOK_RETRIES=5
```

### Logging

The script logs to stderr. To capture logs when run by mdadm:

1. Create a wrapper script:
   ```bash
   #!/bin/bash
   /usr/bin/mdadm-discord-webhook "$@" 2>> /var/log/mdadm-discord.log
   ```

2. Use the wrapper in mdadm.conf:
   ```
   PROGRAM /usr/local/bin/mdadm-discord-wrapper
   ```

## Troubleshooting

### Common Issues

1. **"Permission denied" errors**
   - Ensure the script is executable: `sudo chmod +x /usr/bin/mdadm-discord-webhook`
   - Check file ownership: `sudo chown root:root /usr/bin/mdadm-discord-webhook`

2. **"curl: command not found"**
   ```bash
   # Debian/Ubuntu
   sudo apt-get install curl
   
   # RHEL/CentOS
   sudo yum install curl
   ```

3. **Webhook not receiving messages**
   - Test the webhook URL directly
   - Check Discord server permissions
   - Verify the webhook URL format

4. **mdadm not calling the script**
   - Verify mdadm monitoring is running: `sudo systemctl status mdmonitor`
   - Check mdadm.conf syntax: `sudo mdadm --config-file=/etc/mdadm.conf --detail --scan`
   - Test monitoring manually: `sudo mdadm --monitor --scan --verbose --test`
   - Check system logs: `sudo journalctl -u mdmonitor -f`

### Testing

Test the script manually:

```bash
# Test basic functionality (after configuring webhook URL in script)
sudo mdadm-discord-webhook "TestMessage" "/dev/md0"

# Test with component device
sudo mdadm-discord-webhook "Fail" "/dev/md0" "/dev/sda1"

# Test configuration validation (should show error if webhook not configured)
sudo mdadm-discord-webhook "TestMessage" "/dev/md0"
```

### Debug Mode

Enable verbose logging by modifying the script:

```bash
# Add this line after the shebang in mdadm-discord-webhook.sh
set -x  # Enable debug mode
```

## Uninstallation

To remove the script:

```bash
sudo rm -f /usr/bin/mdadm-discord-webhook
```

Don't forget to remove the `PROGRAM` line from your mdadm.conf file.

## Security Considerations

- **Webhook URL Protection**: Keep your Discord webhook URL private
- **File Permissions**: The script runs as root, ensure proper permissions
- **Network Security**: Consider firewall rules for outbound HTTPS traffic
- **Log Rotation**: If logging to files, implement log rotation

## License

This project is released under the MIT License. See the LICENSE file for details.

## Support

For issues and questions:
1. Check the [Troubleshooting](#troubleshooting) section
2. Review mdadm documentation: `man 5 mdadm.conf`
3. Open an issue in this repository

---

**⚠️ Important**: Always test your configuration in a safe environment before deploying to production systems. RAID failures can result in data loss.

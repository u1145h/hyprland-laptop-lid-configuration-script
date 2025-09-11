# Enable Proper Laptop Lid Function

## Step 1: Disable systemd's default lid handling

1. Open the systemd logind configuration file:

bash

```bash
sudo nano /etc/systemd/logind.conf
```

2. Find and modify these lines (uncomment them by removing `#` and set values):

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

3. Save and exit (Ctrl+X, then Y, then Enter)
4. Restart the logind service:

bash

```bash
sudo systemctl restart systemd-logind
```

## Step 2: Create the lid handler script

1. Create the script file:

bash

```bash
sudo nano /usr/local/bin/lid-handler.sh
```

2. Paste this content:

bash

```bash
#!/bin/bash

# Function to check if on AC power
is_on_ac_power() {
    for adapter in /sys/class/power_supply/A{C,DP}*; do
        [ -r "$adapter/online" ] && [ "$(cat "$adapter/online")" = "1" ] && return 0
    done
    return 1
}

LAPTOP_DISPLAY="eDP-1"
LID_STATE=$(cat /proc/acpi/button/lid/*/state 2>/dev/null | awk '{print $2}')

if is_on_ac_power; then
    # On AC power
    if [ "$LID_STATE" = "closed" ]; then
        # Disable laptop display
        hyprctl keyword monitor "$LAPTOP_DISPLAY,disable"
    else
        # Enable laptop display at 144Hz for performance
        hyprctl keyword monitor "$LAPTOP_DISPLAY,1920x1080@144,0x0,1.25"
    fi
else
    # On battery
    if [ "$LID_STATE" = "closed" ]; then
        # Suspend system
        systemctl suspend
    fi
fi
```

3. Save and exit (Ctrl+X, then Y, then Enter)
4. Make the script executable:

bash

```bash
sudo chmod +x /usr/local/bin/lid-handler.sh
```

## Step 3: Install and configure ACPI daemon

1. Install acpid (if not already installed):

bash

```bash
# For Arch Linux
sudo pacman -S acpid

# For Ubuntu/Debian
sudo apt install acpid

# For Fedora
sudo dnf install acpid
```

2. Create the ACPI event configuration:

bash

```bash
sudo nano /etc/acpi/events/lid
```

3. Add this content:

```
event=button/lid.*
action=/usr/local/bin/lid-handler.sh
```

4. Save and exit (Ctrl+X, then Y, then Enter)

## Step 4: Enable and start ACPI service

1. Enable ACPI service to start at boot:

bash

```bash
sudo systemctl enable acpid
```

2. Start the service immediately:

bash

```bash
sudo systemctl start acpid
```

3. Check if the service is running:

bash

```bash
sudo systemctl status acpid
```

You should see "active (running)" in green.

## Step 5: Test the setup

### Test 1: Check ACPI events

1. Open a terminal and run:

bash

```bash
sudo acpi_listen
```

2. Close and open your laptop lid - you should see events like:

```
button/lid LID close
button/lid LID open
```

3. Press Ctrl+C to stop monitoring

### Test 2: Test the script manually

1. Run the script directly:

bash

```bash
/usr/local/bin/lid-handler.sh
```

2. Check your monitor status:

bash

```bash
hyprctl monitors
```

### Test 3: Test power state detection

1. Check if power detection works:

bash

```bash
# While on AC power, run:
ls /sys/class/power_supply/A*/online
cat /sys/class/power_supply/A*/online

# Should show 1 when plugged in, 0 when on battery
```

### Test 4: Test lid state detection

1. Check lid state detection:

bash

```bash
cat /proc/acpi/button/lid/*/state
```

## Step 6: Optional - Add manual testing binds to Hyprland

1. Open your Hyprland config:

bash

```bash
nano ~/.config/hypr/hyprland.conf
```

2. Add these test binds (optional):

```
# Test binds for lid functionality
bind = $mainMod SHIFT, L, exec, /usr/local/bin/lid-handler.sh
bind = $mainMod SHIFT, D, exec, hyprctl keyword monitor "eDP-1,disable"
bind = $mainMod SHIFT, E, exec, hyprctl keyword monitor "eDP-1,1920x1080@144,0x0,1.25"
```

3. Reload Hyprland config:

bash

```bash
hyprctl reload
```

## Step 7: Troubleshooting

### If events aren't working:

1. Check ACPI events:

bash

```bash
sudo journalctl -u acpid -f
```

2. Check if lid events exist:

bash

```bash
ls /proc/acpi/button/lid/
```

3. Restart ACPI service:

bash

```bash
sudo systemctl restart acpid
```

### If power detection fails:

1. Find your power supply path:

bash

```bash
ls /sys/class/power_supply/
```

2. Check which one shows online status:

bash

```bash
find /sys/class/power_supply/ -name "online" -exec echo {} \; -exec cat {} \;
```

### If Hyprland commands fail:

1. Make sure you're in a Hyprland session
2. Check if `HYPRLAND_INSTANCE_SIGNATURE` is set:

bash

```bash
echo $HYPRLAND_INSTANCE_SIGNATURE
```

## Step 8: Verify everything works

1. **Test on AC power:**
    - Plug in your charger
    - Close the lid → Display should turn off
    - Open the lid → Display should turn back on at 144Hz
2. **Test on battery:**
    - Unplug your charger
    - Close the lid → System should suspend
    - Open the lid → System should wake up

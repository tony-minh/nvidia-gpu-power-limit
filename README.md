
# NVIDIA GPU Power Limit at Boot (Dual RTX 3090)

A simple, persistent setup to limit NVIDIA 3090 power to a sensible efficiency point (default: 280 W) on Proxmox/Linux, surviving reboots via systemd.

This setup:
- Waits for the NVIDIA driver to be ready.
- Enables persistence mode on all GPUs.
- Sets a fixed power limit per GPU.
- Uses a systemd unit so it runs on every boot.
- Requires nvidia-smi installed and working

## Default Settings

- Target: **280 W per GPU**
- Works with any number of NVIDIA GPUs (tested with dual 3090)
- Can be adjusted by changing `TARGET_WATTS`

## Files

1. Script: `/usr/local/bin/set-nvidia-power-limit.sh`
2. Systemd unit: `/etc/systemd/system/set-nvidia-power-limit.service`

---

## 1. Install the Script

Create the script:

```bash
sudo tee /usr/local/bin/set-nvidia-power-limit.sh > /dev/null << 'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

TARGET_WATTS=280
NVIDIA_SMI=/usr/bin/nvidia-smi

for i in {1..30}; do
  if "$NVIDIA_SMI" -L >/dev/null 2>&1; then
    break
  fi
  sleep 1
done

if ! "$NVIDIA_SMI" -L >/dev/null 2>&1; then
  echo "No NVIDIA GPUs found, exiting."
  exit 0
fi

"$NVIDIA_SMI" -pm 1 || true

for idx in $("$NVIDIA_SMI" --query-gpu=index --format=csv,noheader); do
  echo "Setting GPU $idx power limit to ${TARGET_WATTS}W"
  "$NVIDIA_SMI" -i "$idx" -pl "$TARGET_WATTS"
done
SCRIPT
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/set-nvidia-power-limit.sh
```

Edit `TARGET_WATTS` in the script if you want a different limit (e.g. 300 W instead of 280 W).

---

## 2. Install the Systemd Service

Create the service file:

```bash
sudo tee /etc/systemd/system/set-nvidia-power-limit.service > /dev/null << 'SERVICE'
[Unit]
Description=Set NVIDIA GPU power limits at boot
After=multi-user.target
Wants=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/set-nvidia-power-limit.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICE
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable set-nvidia-power-limit.service
sudo systemctl start set-nvidia-power-limit.service
```

---

## 3. Verify

After boot (or after starting the service manually), check the power limit:

```bash
nvidia-smi -q -d POWER | grep -A5 "GPU Power Readings"
```

Or for a quick check of current power state:

```bash
nvidia-smi
```

---

## Adjusting the Power Limit

To change the limit:

1. Edit the script:

   ```bash
   sudo nano /usr/local/bin/set-nvidia-power-limit.sh
   ```

2. Change this line:

   ```bash
   TARGET_WATTS=280
   ```

   to your desired value (e.g. `TARGET_WATTS=300` or `TARGET_WATTS=250`).

3. Restart the service:

   ```bash
   sudo systemctl restart set-nvidia-power-limit.service
   ```

---

## Notes

- This is designed for headless Proxmox / Linux boxes.
- Persistence mode (`-pm 1`) keeps the driver and GPU state stable.
- For dual RTX 3090s, 280 W gives good efficiency with minimal performance loss in LLM inference.
```

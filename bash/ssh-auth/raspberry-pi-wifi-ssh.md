# Raspberry Pi: check Wi-Fi, reconnect, SSH in
<!-- verified: 2026-08 -->

## Check which Wi-Fi network the Pi thinks it's on
```
iwgetid
```

## Force it to re-read the Wi-Fi config and reconnect
```
sudo wpa_cli -i <INTERFACE> reconfigure
```
Example:
```
sudo wpa_cli -i wlan0 reconfigure
```
**Why it bites:** `reconfigure` only picks up changes if the config file below was already edited — running it without editing anything just reconnects to the same network and looks like it "did nothing," because it did nothing.

## Edit the Wi-Fi config directly
```
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

## SSH into the Pi by hostname (mDNS, no IP needed)
```
ssh <USER>@<HOSTNAME>.local
```
Example:
```
ssh pi@raspberrypi.local
```
**Why it bites:** `.local` resolution depends on mDNS/Bonjour on the *client* machine, not just the Pi. If it suddenly stops resolving after working fine before, it's usually the laptop's network switching (e.g. a VPN toggling) breaking mDNS, not the Pi being down — `ping raspberrypi.local` before assuming the Pi died.

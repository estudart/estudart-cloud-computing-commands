# Wi-Fi Hotspot commands with nmcli

# Create and start a hotspot
sudo nmcli dev wifi hotspot ifname wlan0 con-name "RaspbotHotspot" ssid "RaspbotHotspot" password "PASSWORD"

# Start the hotspot
sudo nmcli con up "RaspbotHotspot"

# Stop the hotspot
sudo nmcli con down "RaspbotHotspot"

# Change the hotspot name
sudo nmcli con mod "RaspbotHotspot" wifi.ssid "NEW_HOTSPOT_NAME"

# Change the hotspot password
sudo nmcli con mod "RaspbotHotspot" wifi-sec.psk "NEW_PASSWORD"

# Use a fixed hotspot IP address
sudo nmcli con mod "RaspbotHotspot" ipv4.method shared ipv4.addresses "10.42.0.1/24"

# Start the hotspot automatically after reboot
sudo nmcli con mod "RaspbotHotspot" \
  connection.autoconnect yes \
  connection.interface-name wlan0

# Show the hotspot password
sudo nmcli dev wifi show-password ifname wlan0

# Delete the hotspot
sudo nmcli con delete "RaspbotHotspot"

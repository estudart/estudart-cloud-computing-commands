# Wi-Fi commands with nmcli

# List devices
nmcli dev

# List available Wi-Fi networks
nmcli dev wifi

# Connect and save the network
sudo nmcli dev wifi connect "SSID" password "PASSWORD"

# Connect using wlan1
sudo nmcli dev wifi connect "SSID" password "PASSWORD" ifname wlan1

# Connect without saving the password in shell history
sudo nmcli --ask dev wifi connect "SSID" ifname wlan1

# List saved connections
nmcli con show

# Update an existing Wi-Fi password
sudo nmcli con mod "CONNECTION_NAME" wifi-sec.psk "NEW_PASSWORD"

# Reconnect
sudo nmcli con up "CONNECTION_NAME"

# Delete a saved connection
sudo nmcli con delete "CONNECTION_NAME"

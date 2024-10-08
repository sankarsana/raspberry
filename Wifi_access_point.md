## Install required packages
```sudo apt-get install hostapd dnsmasq```
## Hostapd
- The purpose of Hostapd is to set WiFi as an access point
- we need to write a new config file for hostapd
    ```
     sudo vim /etc/hostapd/hostapd.conf
    ```
    ```
    interface=wlan0
    driver=nl80211
    ssid=Sankarsana_rasp
    hw_mode=g
    channel=7
    wmm_enabled=0
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=00000000
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
    ```
- Then we need to tell hostapd to use our config file, edit `/etc/default/hostapd` and change the line starts with `#DAEMON_CONF`, remember to remove `#`

    ```
    DAEMON_CONF="/etc/hostapd/hostapd.conf"
    ```
- Then Let's start hostapd
    ```
    sudo systemctl unmask hostapd &&
    sudo systemctl enable hostapd &&
    sudo systemctl start hostapd
    ```
## dnsmasq
- The purpose of dnsmasq is to act as DHCP Server, so when a devies connects to Raspberry Pi it can get an IP assigned to it.

- make a backup of default config by:
    ```
    sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
    ```

- Create a new config file by:
    ```
    sudo vim /etc/dnsmasq.conf
    ```

- This config file will automatically assign addresses between `192.168.1.2` and `192.168.1.199` with lease time `24` hours.

    ```
    interface=wlan0
    dhcp-range=192.168.1.2,192.168.1.199,255.255.255.0,24h
    ```
- Uncomment `#port=5353`
- Uncomment and change the following line to specify the DNS servers to be used to resolve host names
    ```
    dhcp-option=6,8.8.8.8,8.8.4.4
    ```
- Then Let's reload dnsmasq config
    ```
    sudo systemctl unmask dnsmasq &&
    sudo systemctl enable dnsmasq &&
    sudo systemctl start dnsmasq
    ```
## Enabling IP forwarding
- Edit file:
    ```
    sudo vim /etc/sysctl.conf
    ```
- Find and unkoment `net.ipv4.ip_forward=1`
- Apply changes:
    ```
    sudo sysctl -p
    ```
## Setting up iptables for NAT
- Enable NAT on `eth0` interface
    ```
    sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    sudo iptables -t nat -A POSTROUTING -o ppp0 -j MASQUERADE
    ```
This setting will disappear after reboot.
### Setup the script at boot.
- Create a systemd service file
    ```
    sudo vim /etc/systemd/system/auto-run.service
    ```
- Content
    ```
    [Unit]
    Description=Auto run
    After=network.target
    
    [Service]
    ExecStart=/etc/auto-run.sh
    
    [Install]
    WantedBy=multi-user.target
    ```
- Create script file
    ```
    sudo vim /etc/auto-run.sh
    ```
- Content
    ```
    #!/bin/bash
    
    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    iptables -t nat -A POSTROUTING -o ppp0 -j MASQUERADE
    ```
- Apply exec permission
    ```
    sudo chmod +x /etc/auto-run.sh
    ```
- Enable the service:
    ```
    sudo systemctl daemon-reload &&
    sudo systemctl enable auto-run.service &&
    sudo systemctl start auto-run.service
    ```
## Solving startup Error:
***You don't have to do this. It worked for me without it.***
- On System startup, dnsmasq will not wait for wlan0 interface to initialize and will fail with error `wlan0 not found`.

- We need to tell systemd to launch it after network get ready, so we will modify dnsmasq service file by adding `After=` and `Wants=` under `[Unit]` section.

    ```
    sudo vim /lib/systemd/system/dnsmasq.service
    ```

    ```
    [Unit]
    ...
    After=network-online.target
    Wants=network-online.target
    ```
## Config static IP
- Ubuntu uses cloud-init for initial setup, so will modify the following file to set wlan0 IP.
- `DON'T USE TABS IN THIS FILE, IT WILL NOT WORK, EVER!!`
- Modify the cloud-init file by
    ```
    sudo vim /etc/netplan/50-cloud-init.yaml
    ```
- Add the following content to the file:
    ```
            wlan0:
                dhcp4: false
                addresses:
                - 192.168.1.1/24
                nameservers:
                    addresses: [8.8.8.8, 8.8.4.4]
    ```
    
- The file will finally looks like this:
    ```
    # This file is generated from information provided by
    # the datasource.  Changes to it will not persist across an instance.
    # To disable cloud-init's network configuration capabilities, write a file
    # /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
    # network: {config: disabled}
    network:
        version: 2
        ethernets:
            eth0:
                dhcp4: true
                match:
                    macaddress: 12:34:56:78:ab:cd
                set-name: eth0
            wlan0:
                dhcp4: false
                addresses:
                - 192.168.1.1/24
                nameservers:
                    addresses: [8.8.8.8, 8.8.4.4]
    ```
## Finally:
- Reboot your Raspberry Pi and check if you can connect to it over WiFi and can SSH.

## Notes:
- if you can't see Raspberry Pi Hot spot then `hostapd` is not working, you can check its logs by `sudo systemctl status hostapd`.

- if you can coonect to Raspberry Pi but can't get an IP then `dnsmasq` is not working, you can check its logs by `sudo systemctl status dnsmasq`.

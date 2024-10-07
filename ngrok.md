# Установка ngrok
Установка через apt https://dashboard.ngrok.com/get-started/setup/raspberrypi
```
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
	| sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
	&& echo "deb https://ngrok-agent.s3.amazonaws.com buster main" \
	| sudo tee /etc/apt/sources.list.d/ngrok.list \
	&& sudo apt update \
	&& sudo apt install ngrok
```

## Настройка
~/.config/ngrok/ngrok.yml
```
version: "3"
agent:
    authtoken: 7uSbB78xqYkghb8fxJUPU_4nXusfFBjTen6cQZaZNin
tunnels:
    ssh:
        proto: tcp
        addr: 22
```

## Service
/etc/systemd/system/ngrok.service
```
[Unit]
Description=ngrok
After=network.target
After=syslog.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ngrok start --all --config /home/san/.config/ngrok/ngrok.yml
Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

api token
`2mwMjhFD7APvtAEnTPvXsksAByP_5RDuzWSR6UUToV4LfbdnB`

## скрипт для поключению по ssh
```
#!/bin/bash

res=$(curl --location --request GET 'https://api.ngrok.com/tunnels' \
--header 'Authorization: Bearer 2mwMjhFD7APvtAEnTPvXsksAByP_5RDuzWSR6UUToV4LfbdnB' \
--header 'Ngrok-Version: 2' | awk 'BEGIN { FS="\""; RS="," }; { if ($2 == "public_url") {print $4}}')

if [ -z "$res" ]
then
    echo "No tunnels"
    exit
fi

ip_port=$(echo ${res:6})

arr=($(echo $ip_port | tr ":" "\n"))

ip=$(echo ${arr[0]})
port=$(echo ${arr[1]})

ssh san@$ip -p $port
```

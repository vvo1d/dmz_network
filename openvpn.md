# OpenVPN Setup on Red OS

This guide sets up an OpenVPN server and client, including certificate generation and network configuration.

## Server Setup

# Set SELinux to permissive mode

sudo sed -i 's/enforcing/permissive/g' /etc/selinux/config

sudo setenforce 0

# Install OpenVPN and easy-rsa

sudo dnf install -y openvpn easy-rsa

# Create vars file for certificate generation

sudo nano /usr/share/easy-rsa/3/vars

# Contents

export KEY_COUNTRY="RU"

export KEY_PROVINCE="Spb"

export KEY_CITY="Spb"

export KEY_ORG="TKUiK"

export KEY_EMAIL="admin@example.com"

export KEY_CN="openvpnserver"

export KEY_OU="ITotdel"

export KEY_NAME="vpn.example.com"

export KEY_ALTNAMES="myopenvpn"

# Generate certificates and keys

cd /usr/share/easy-rsa/3

sudo ./easyrsa init-pki

sudo ./easyrsa build-ca # Passphrase: xxXX1234 (example)

sudo ./easyrsa gen-crl

sudo ./easyrsa gen-dh

sudo ./easyrsa gen-req vpn-server nopass

sudo ./easyrsa sign-req server vpn-server # Enter 'yes' for details sudo openvpn --genkey secret pki/ta.key

# Copy certificates to OpenVPN directory

sudo mkdir -p /etc/openvpn/server/keys

sudo cp pki/ca.crt pki/issued/vpn-server.crt pki/private/vpn-server.key pki/dh.pem pki/ta.key pki/crl.pem /etc/openvpn/server/keys

# Configure the server

sudo nano /etc/openvpn/server/server.conf

# Contents

local 192.168.0.26

port 1194

proto udp

dev tun

ca keys/ca.crt

cert keys/vpn-server.crt

key keys/vpn-server.key

dh keys/dh.pem

tls-auth keys/ta.key 0

ifconfig 10.1.1.10 255.255.255.0

ifconfig-pool 10.1.1.3 10.1.1.254

push "redirect-gateway def1"

push "route-gateway 10.1.1.1"

push "dhcp-option DNS 8.8.8.8"

ifconfig-pool-persist ipp.txt

keepalive 10 120

max-clients 15

client-to-client

persist-key

persist-tun

status /var/log/openvpn/openvpn-status.log

log-append /var/log/openvpn/openvpn.log

verb 0

mute 20

daemon

mode server

tls-server

comp-lzo yes

cipher AES-256-GCM

data-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC:AES-128-CBC

crl-verify /etc/openvpn/server/keys/crl.pem

# Create log directory

sudo mkdir /var/log/openvpn

# Configure NAT and iptables

iptables -t nat -A POSTROUTING -s 10.1.1.0/24 -o ens33 -j MASQUERADE

iptables -A FORWARD -i tun0 -o ens33 -j ACCEPT

iptables -A FORWARD -i ens33 -o tun0 -m state --state RELATED,ESTABLISHED -j ACCEPT

iptables -t nat -A POSTROUTING -o ens33 -s 10.0.0.0/24 -j MASQUERADE

iptables -A FORWARD -d vk.com -j REJECT

iptables -A FORWARD -d ok.ru -j REJECT

iptables -A FORWARD -d mail.ru -j REJECT

iptables-save &gt; /etc/sysconfig/iptables

# Enable IP forwarding

sudo nano /etc/sysctl.conf

# Add

net.ipv4.ip_forward=1

# Apply

sudo sysctl -p /etc/sysctl.conf

# Enable and start OpenVPN server

sudo systemctl enable openvpn-server@server

sudo systemctl start openvpn-server@server

# Restore SELinux to enforcing mode

sudo sed -i 's/permissive/enforcing/g' /etc/selinux/config sudo setenforce 1

## Client Setup

# Generate client certificates

cd /usr/share/easy-rsa/3

sudo ./easyrsa gen-req client1 nopass

sudo ./easyrsa sign-req client client1 # Enter 'yes' for confirm

# Copy certificates to a temporary directory

sudo mkdir /tmp/keys

sudo cp pki/issued/client1.crt pki/private/client1.key pki/dh.pem pki/ca.crt pki/ta.key /tmp/keys

sudo chmod -R a+r /tmp/keys

# Create base client configuration

sudo mkdir /etc/openvpn/keyovpn cd /etc/openvpn/keyovpn 

sudo nano /etc/openvpn/keyovpn/base-client.conf

# Contents

client 

resolv-retry infinite 

nobind 

remote 192.168.0.26 1194 

proto udp 

dev tun 

comp-lzo yes 

tls-client 

key-direction 1 

float 

keepalive 10 120 

persist-key 

persist-tun 

verb 0 

cipher AES-256-GCM

# Create script to generate .ovpn file

sudo nano /etc/openvpn/keyovpn/make_config.sh

# Contents

#!/bin/bash

KEY_DIR=/tmp/keys

OUTPUT_DIR=/etc/openvpn/keyovpn/

# and other base client configuration

BASE_CONFIG=./base-client.conf

cat ${BASE_CONFIG} \
&lt;(echo ) \
&lt;(echo -e '') \
${KEY_DIR}/ca.crt \
&lt;(echo -e '\\n') \
${KEY_DIR}/${1}.crt \
&lt;(echo -e '\\n') \
${KEY_DIR}/${1}.key \
&lt;(echo -e '\\n') \
${KEY_DIR}/ta.key \
&lt;(echo -e '') \\

&gt; ${OUTPUT_DIR}/${1}.ovpn

# Make the script executable and run it

sudo chmod +x make_config.sh 

sudo ./make_config.sh client1

# Transfer the .ovpn file to the client (machine B)

scp /etc/openvpn/keyovpn/client1.ovpn b@192.168.0.27:/home/b

# Set up auto-start for the server

echo "@reboot root systemctl start openvpn-server@server" &gt;&gt; /etc/crontab

## Client Configuration (Machine B)

# Install OpenVPN client

sudo dnf install -y NetworkManager-openvpn-gnome

# Disable external network access, allow only 10.0.0.0/24 subnet

# Import the .ovpn file on the client

# Set up auto-start

echo "@reboot root systemctl start openvpn@client" &gt;&gt; /etc/crontab

# Note: Machine B routes all traffic through Machine A (the OpenVPN server)

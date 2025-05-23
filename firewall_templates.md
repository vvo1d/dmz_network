# Firewall Templates for Red OS (iptables)

This guide provides iptables rules for different network scenarios, including OpenVPN server, service server, and client configurations.

## OpenVPN Server (192.168.88.149)

# Goal:

# - Allow VPN clients internet access via NAT

# - Allow VPN access to the service network (192.168.10.0/24)

# - Block direct internet access to internal networks

# Interfaces:

# - External: ens33 (IP: 192.168.88.149)

# - VPN: tun0 (Subnet: 10.1.1.0/24)

# - Internal (services): eth1 (Subnet: 192.168.10.0/24)

# Reset rules

iptables -F iptables -t nat -F iptables -X iptables -t nat -X

# Default policies

iptables -P INPUT DROP iptables -P FORWARD DROP iptables -P OUTPUT ACCEPT

# Allow local traffic

iptables -A INPUT -i lo -j ACCEPT

# Allow established connections

iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT 

iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Allow OpenVPN (port 1194/UDP)

iptables -A INPUT -i ens33 -p udp --dport 1194 -j ACCEPT

# NAT for VPN clients

iptables -t nat -A POSTROUTING -o ens33 -s 10.1.1.0/24 -j MASQUERADE

# Allow VPN to service network

iptables -A FORWARD -i tun0 -o eth1 -s 10.1.1.0/24 -d 192.168.10.0/24 -j ACCEPT 

iptables -A FORWARD -i eth1 -o tun0 -s 192.168.10.0/24 -d 10.1.1.0/24 -j ACCEPT

# Allow VPN clients internet access

iptables -A FORWARD -i tun0 -o ens33 -s 10.1.1.0/24 -j ACCEPT 

iptables -A FORWARD -i ens33 -o tun0 -d 10.1.1.0/24 -j ACCEPT

# Block internet access to service network

iptables -A FORWARD -i ens33 -o eth1 -d 192.168.10.0/24 -j DROP

# Protect against attacks

iptables -A INPUT -m conntrack --ctstate INVALID -j DROP 

iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT

# Enable IP forwarding

echo 1 &gt; /proc/sys/net/ipv4/ip_forward 

echo "net.ipv4.ip_forward=1" &gt;&gt; /etc/sysctl.conf

## Service Server (192.168.10.0/24)

# Goal:

# - Allow access only from 192.168.88.0/24 (OpenVPN) and 10.1.1.0/24 (VPN)

# - Block all other connections

# Reset rules

iptables -F iptables -X

# Default policies

iptables -P INPUT DROP iptables -P FORWARD DROP iptables -P OUTPUT ACCEPT

# Allow local traffic

iptables -A INPUT -i lo -j ACCEPT

# Allow established connections

iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Allow access from trusted networks

iptables -A INPUT -s 192.168.88.0/24 -j ACCEPT iptables -A INPUT -s 10.1.1.0/24 -j ACCEPT

# Allow HTTP (port 80)

iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Drop all other traffic

iptables -A INPUT -j DROP

## 

# Verify:

# - All traffic goes through VPN

curl ifconfig.me # Should show OpenVPN server IP

# - Access to services

curl http://192.168.10.100 # Should work

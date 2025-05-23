# DNS-over-TLS (DoT) Setup with BIND and Unbound on Red OS

This guide configures BIND as an authoritative DNS server and Unbound as a recursive resolver with DNS-over-TLS (DoT).

## Install BIND and Unbound

sudo yum install bind bind-utils unbound\*

## Configure BIND

# Edit /etc/named.conf

# Replace the options section

options { 

    recursion no; // Disable recursion (BIND is authoritative only) 

};

# Add zone configuration at end of file

zone "irwol.ru" { 

    type master; 

    file "/var/named/irwol.ru.zone"; 

    allow-update { none; }; 

};

# Create zone file

sudo nano /var/named/irwol.ru.zone

# Contents

$TTL 86400 

@ IN SOA dc1.irwol.ru. admin.irwol.ru. ( 

    2024051901 ; Serial 

    3600 ; Refresh 

    1800 ; Retry 

    604800 ; Expire 

    86400 ; Minimum TTL 

)

@ IN NS dc1.irwol.ru. 

@ IN MX 10 dc1.irwol.ru. 

dc1 IN A 127.0.0.1

# Verify and start BIND

sudo named-checkconf 

sudo named-checkzone irwol.ru /var/named/irwol.ru.zone 

sudo systemctl enable named 

sudo systemctl restart named

## Configure Unbound

# Edit /etc/unbound/unbound.conf

# Add the following

server: 

    interface: 127.0.0.1@5353 

    access-control: 127.0.0.0/8 allow 

    do-ip4: yes 

    do-ip6: no 

    do-udp: yes 

    do-tcp: yes 

    tls-cert-bundle: /etc/pki/tls/certs/ca-bundle.crt # Path to root certificates 

    do-not-query-localhost: no # Allow queries to localhost

forward-zone: 

    name: "." 

    forward-tls-upstream: yes 

\# Cloudflare DNS-over-TLS 

    forward-addr: 1.1.1.1@853#cloudflare-dns.com 

    forward-addr: 1.0.0.1@853#cloudflare-dns.com 

\# Google DNS-over-TLS 

    forward-addr: 8.8.8.8@853#dns.google 

    forward-addr: 8.8.4.4@853#dns.google

stub-zone: 

    name: "irwol.ru" 

    stub-addr: 127.0.0.1@53 # Point to BIND for local domain

# Set the machine as a DNS server

sudo nano /etc/resolv.conf

# Replace with

nameserver 127.0.0.1 

nameserver 8.8.8.8 

sudo chattr +i /etc/resolv.conf

# Allow DNS queries in the firewall

sudo firewall-cmd --add-service=dns –permanent 

sudo firewall-cmd --add-port=5353/tcp --permanent 

sudo firewall-cmd --reload

# Verify and start Unbound

sudo unbound-checkconf 

sudo systemctl enable unbound 

sudo systemctl restart unbound

## Test the Configuration

# Check local domain resolution (via BIND)

dig @127.0.0.1 dc1.irwol.ru

# Check external resolution (via Unbound with DoT)

dig @127.0.0.1 google.com

# Expected output includes SERVER: 127.0.0.1#53(127.0.0.1)

# Check DoT traffic

sudo tcpdump -i any port 853 -vv

# Look for TLS traffic on port 853

# Additional DoT verification

dig @127.0.0.1 -p 5353 google.com # Through Unbound (DoT) 

dig @127.0.0.1 dc1.irwol.ru # Through BIND

# Check Unbound logs

journalctl -u unbound -f

## Verify DoT Traffic

# Key indicators of DoT

# - Traffic on port 853 (e.g., 1.1.1.1:853 for Cloudflare)

# - TLS handshake visible (SYN, SYN-ACK, PUSH flags)

# - Larger packet sizes due to TLS overhead

# Verify certificates

openssl s_client -connect 1.1.1.1:853 -servername cloudflare-dns.com

# Expected: Cloudflare certificate and successful handshake (Verify return code: 0)

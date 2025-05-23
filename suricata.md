# Suricata IDS/IPS Setup on Red OS

This guide configures Suricata as an Intrusion Detection and Prevention System (IDS/IPS).

## Initial Setup

# Follow the Red OS guide for initial Suricata setup

# Configure netmap parameters

set suricata netmap parameters copy-mode ips

set suricata netmap parameters checksum-checks yes

set suricata enable no commit

set suricata enable yes commit

set system option performance latency # Suitable for small networks; use 'throughput' for enterprise

## Create Custom Rules

sudo mkdir /etc/suricatarules

sudo nano /etc/suricatarules/VON.rules

# Contents

# Block ICMP echo requests (e.g., ping)

drop icmp any any -&gt; any any (msg:"Blocked ICMP Echo Request"; sid:1000001; rev:1;)

# Block SMB exploits on port 445

drop tcp any any -&gt; any 445 (msg:"Block SMB Exploits"; flow:to_server; sid:1000002; rev:1;)

# Detect TCP SYN scans

alert tcp any any -&gt; any any (flags:S; msg:"TCP SYN Scan Detected"; sid:1000003; rev:1;)

# Monitor TCP connections to port 80

alert tcp any any -&gt; any 80 (msg:"TCP Connection to Port 80"; sid:1000004; rev:1;)

# Alternative rule for external connections to port 80

alert tcp !\[192.168.22.0/24\] any -&gt; any 80 (msg:"External TCP to Port 80"; sid:1000005;)

# Optional: Block SSH brute-force attempts

drop tcp any any -&gt; any 22 (msg:"SSH Bruteforce Block"; sid:1000005; rev:1;)

## Configure Suricata for IPS with NFQ

# Edit /etc/suricata/suricata.yaml

af-packet:

```
- interface: eth0

  threads: auto

  cluster-type: cluster_flow

  defrag: yes

  use-mmap: yes

  ring-size: 200000

  buffer-size: 64535

  checksum-checks: auto
```

outputs:

  - eve-log:

      enabled: yes

      filetype: regular

      filename: eve.json

      types:

        - alert:

            payload: yes

            payload-printable: yes

            packet: yes

            http-body: yes

            http-body-printable: yes

            metadata: yes

        - http:

            extended: yes

        - dns:

            query: yes

            answer: yes

        - tls:

            extended: yes

# Configure NFQ

sudo iptables -I FORWARD -j NFQUEUE --queue-num 0 

sudo iptables -I INPUT -j NFQUEUE --queue-num 0 

sudo iptables -I OUTPUT -j NFQUEUE --queue-num 0

nfq:

  mode: accept

  fail-open: yes

  qids: \[ 0 \]

  queues: \[0\]

  threads: 2

  max-pending-packets: 1024

  defrag: yes

# Specify rule file

rule-files:

    - /etc/suricatarules/VON.rules

## Start Suricata

sudo suricata -c /etc/suricata/suricata.yaml -s /etc/suricatarules/VON.rules -q 0 --runmode=workers

## Test Rules

# Test rule 1 (ICMP block)

ping \[suricata_ip\]

# Should be blocked; check logs

tail /var/log/suricata/fast.log -n 5

# Test rule 2 (SMB block)

nmap -p 445 \[suricata_ip\]

# Expected: State: filtered

tail /var/log/suricata/fast.log -n 5

# Test rule 3 (SYN scan detection)

sudo nmap -sS \[suricata_ip\]

# Test rule 4 (HTTP connection)

curl http://\[suricata_ip\]

# Fail2Ban Setup on Red OS

This guide sets up Fail2Ban to protect services like Roundcube (used with iRedMail) from brute-force attacks.

## Configuration

# Note: Detailed Fail2Ban setup is available in the Red OS knowledge base

# Add the following configuration for Roundcube protection

\[roundcube-iredmail\] 

enabled = true 

filter = roundcube-iredmail 

logpath = /var/log/nginx/roundcube.error.log 

maxretry = 3 

findtime = 3600 

bantime = 86400 

action = firewallcmd-ipset\[port="http,https", protocol="tcp"\]

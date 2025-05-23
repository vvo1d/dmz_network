# Web Application Firewall (WAF) Setup with ModSecurity and NGINX on Red OS

This guide sets up a WAF using ModSecurity with NGINX.

## Initial Setup

# Set SELinux to permissive mode

sudo setenforce 0

# Install dependencies

sudo dnf install yum 

sudo yum update 

sudo yum install -y autoconf automake git pkgconf wget libtool\* gcc\* pcre\* yajl\* lmdb\* libxml2\* ssdeep\* lua-\* geoip\* --skip-broken

# Create NGINX repository file

sudo nano /etc/yum.repos.d/nginx.repo

# Contents

\[nginx-1.13.7\] name=nginx repo 

baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/ 

gpgcheck=0 

enabled=1

# Install NGINX

sudo yum install nginx\*

## Install ModSecurity

cd /home/user/Downloads 

git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity.git 

cd ModSecurity 

git submodule init 

git submodule update 

./build.sh 

./configure 

make 

make install

# Install ModSecurity NGINX connector

git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git

# Download and compile NGINX with ModSecurity module

cd .. 

wget http://nginx.org/download/nginx-1.13.7.tar.gz 

tar zxvf nginx-1.13.7.tar.gz 

cd nginx-1.13.7 

./configure --with-compat --add-dynamic-module=ModSecurity-nginx 

make modules

# Copy the module

cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules

# Configure ModSecurity

sudo mkdir /etc/nginx/modsec 

cd /etc/nginx/modsec 

sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended 

sudo mv modsecurity.conf-recommended modsecurity.conf

# Enable ModSecurity

sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsec/modsecurity.conf

# Download and configure OWASP Core Rule Set

cd /usr/local 

git clone https://github.com/coreruleset/coreruleset cd coreruleset/ 

mv crs-setup.conf.example crs-setup.conf 

cd rules/ 

mv REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf 

mv RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf

# Create main.conf for ModSecurity rules

# Create /etc/nginx/modsec/main.conf

# Contents

# Include the recommended configuration

Include /etc/nginx/modsec/modsecurity.conf Include /usr/local/coreruleset/crs-setup.conf Include /usr/local/coreruleset/rules/\*.conf

# Install additional ModSecurity packages

sudo yum install mod_security mod_security_crs nginx-mod-modsecurity -y

# Configure NGINX to use ModSecurity

sudo nano /etc/nginx/conf.d/modsecurity.conf

# Contents

modsecurity on; 

modsecurity_rules_file /etc/nginx/modsecurity.d/modsecurity.conf;

# Set up ModSecurity directory (HERE CAN BE PROBLEM, just clean file from this)

sudo mkdir /etc/nginx/modsecurity.d 

sudo ln -s /usr/share/mod_security_crs /etc/nginx/modsecurity.d/crs 

echo 'Include /etc/nginx/modsecurity.d/crs/crs-setup.conf' | sudo tee -a /etc/nginx/modsecurity.d/modsecurity.conf 

echo 'Include /etc/nginx/modsecurity.d/crs/rules/\*.conf' | sudo tee -a /etc/nginx/modsecurity.d/modsecurity.conf

# Create whitelist configuration

sudo nano /etc/nginx/modsecurity.d/whitelist.conf

# Contents

# Allow authorization parameters

SecRule ARGS_NAMES "@contains login" "id:1001,phase:1,nolog,allow" 

SecRule ARGS_NAMES "@contains password" "id:1002,phase:1,nolog,allow"

# Disable checks for static files

SecRule REQUEST_URI "@endsWith .css" "id:1003,phase:1,nolog,allow" 

SecRule REQUEST_URI "@endsWith .js" "id:1004,phase:1,nolog,allow"

# Update modsecurity.conf

sudo nano /etc/nginx/modsecurity.d/modsecurity.conf

# Contents

SecRuleEngine On 

SecAuditLog /var/log/nginx/modsec_audit.log 

Include /usr/local/coreruleset/crs-setup.conf 

Include /usr/local/coreruleset/rules/\*.conf 

Include /etc/nginx/modsecurity.d/whitelist.conf

# Configure firewall

sudo firewall-cmd --permanent --add-service=http sudo firewall-cmd --permanent --add-service=https 

sudo firewall-cmd –reload sudo setsebool -P httpd_can_network_connect 1

# Update NGINX configuration

# Edit /etc/nginx/nginx.conf and add at the beginning

include /usr/share/nginx/modules/\*.conf; 

events { 

    ... 

}

http { 

    ... 

    include /etc/nginx/conf.d/\*.conf;

```
server {
    listen       80;
    listen       [::]:80;
    server_name  _;
    root         /usr/share/nginx/html;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;
}
```

}

# Set SELinux to permissive permanently

sudo sed -i 's/enforcing/permissive/g' /etc/selinux/config

# Test and start NGINX

nginx -t 

sudo systemctl start nginx

# Test WAF

curl "http://localhost/?&lt;script&gt;alert(1)&lt;/script&gt;"

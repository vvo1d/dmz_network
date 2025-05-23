# Zabbix Server and Agent Setup on Red OS

This guide sets up Zabbix server and agent on Red OS, including PostgreSQL configuration.

## Zabbix Server Setup

# Note: Detailed setup is available in the Red OS knowledge base

# Configure httpd

# Edit /etc/httpd/conf/httpd.conf to set ServerName and Listen

# Example:

# ServerName ServerName dc1.irwol.ru:8080

# Listen 8080

# Configure Zabbix

# Edit Zabbix.conf to set

DBPort=5432 

EnableGlobalScripts=1

# Update PostgreSQL user password

sudo -u postgres psql 

ALTER USER zabbix WITH PASSWORD 'new_password';

# If PHP version is incompatible

sudo rpm -e --nodeps php 

sudo dnf install php 

sudo dnf install php-bcmath-8.1.\* php-mbstring-8.1.\* php-gd-8.1.\* php-xml-8.1.\* php-ldap-8.1.\* php-pgsql-8.1.\*

## Zabbix Agent Setup

# Edit the agent configuration file

sudo nano /etc/zabbix/zabbix_agentd.conf

# Set the following parameters

Server=IP_address_of_Zabbix_server # Example: 192.168.1.100 

ServerActive=IP_address_of_Zabbix_server # For active checks 

Hostname=Agent_hostname # Must match the agent hostname in Zabbix web interface

## Firewall Configuration

# Install and start firewalld

sudo dnf install firewall\* 

sudo systemctl start firewalld.service

# Allow Zabbix agent port

sudo firewall-cmd --permanent --add-port=10050/tcp 

sudo firewall-cmd –reload

## SELinux Configuration

sudo setsebool -P zabbix_can_network=1

# Restart the agent

sudo systemctl restart zabbix-agent

## Configure Zabbix Web Interface

# In the Zabbix web interface

# Go to Configuration → Hosts → Create host

# Set:

# - Host name: Matches Hostname in zabbix_agentd.conf

# - Templates: Attach "Linux by Zabbix agent" template

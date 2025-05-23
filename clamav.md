# ClamAV Setup on Red OS

This guide installs and configures ClamAV antivirus from source\[clamav.ru\](https://clamav.ru), as specified in the original document.

## Install Dependencies

sudo yum install -y gcc make automake autoconf libtool openssl openssl-devel zlib zlib-devel pcre2 pcre2-devel curl

# If dependencies are missing, use this fallback

sudo dnf install gcc\* openssl\* libss\* --allowerasing --skip-broken

## Download and Extract Source

# Visit https://clamav.ru and download the latest version (e.g., clamav-1.2.0.tar.gz)

tar -xvf clamav-0.103.2.tar.gz 

cd clamav-0.103.2/

## Build and Install

# Configure the build

./configure --prefix=/usr/local/clamav --with-openssl=/usr/include/openssl --with-zlib=/usr/include/zlib

# If OpenSSL is not found, use this

./configure --prefix=/usr/local/clamav --with-zlib=/usr/include/zlib

# Compile and install

make -j$(nproc) 

sudo make install

## Configure ClamAV

# Create user and group

sudo groupadd clamav 

sudo useradd -g clamav -s /bin/false -d /dev/null clamav

# Create directories

sudo mkdir -p /usr/local/clamav/{db,logs,run} 

sudo chown -R clamav:clamav /usr/local/clamav

# Configure clamd.conf

# Edit /usr/local/clamav/etc/clamd.conf with the following

LogFile /usr/local/clamav/logs/clamd.log 

LogTime yes 

DatabaseDirectory /usr/local/clamav/db 

LocalSocket /usr/local/clamav/run/clamd.sock 

User clamav

# Configure freshclam.conf

# Edit /usr/local/clamav/etc/freshclam.conf with the following

DatabaseDirectory /usr/local/clamav/db 

LogFile /usr/local/clamav/logs/freshclam.log 

LogTime yes 

PrivateMirror https://clamav-mirror.ru/

## Update Antivirus Databases

sudo /usr/local/clamav/bin/freshclam

## Start ClamAV

# Start the clamd daemon

sudo /usr/local/clamav/sbin/clamd

# Create a systemd service file for clamd

# Create /etc/systemd/system/clamd.service

\[Unit\] 

Description=ClamAV Daemon 

After=syslog.target network.target

\[Service\] 

Type=simple 

ExecStart=/usr/local/clamav/sbin/clamd 

User=clamav 

Restart=on-failure

\[Install\] 

WantedBy=multi-user.target

# Enable and start the service

sudo systemctl daemon-reload sudo systemctl enable --now clamd

## Test ClamAV

# Create an EICAR test file

echo 'X5O!P%@AP\[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H\*' &gt; /tmp/eicar.txt

# Scan the test file

/usr/local/clamav/bin/clamscan /tmp/eicar.txt

# Expected output: /tmp/eicar.txt: Eicar-Test-Signature FOUND

## Integrate with iRedMail (Optional)

# Edit /etc/amavisd/amavisd.conf to enable ClamAV

$av_scanner = "ClamAV"; 

$av_scanner_backup = undef;

## SELinux and Firewall

# If SELinux is enabled

sudo setsebool -P antivirus_can_scan_system 1 

sudo restorecon -Rv /usr/local/clamav

# Open firewall ports for database updates

sudo firewall-cmd --permanent --add-port=3310/tcp sudo firewall-cmd --reload

## Schedule Database Updates

# Add to /etc/crontab

0 3 \* \* \* /usr/local/clamav/bin/freshclam –quiet

## Additional Directory Setup

# Create missing directory if needed

sudo mkdir -p /usr/local/clamav/share/clamav 

sudo chown -R clamav:clamav /usr/local/clamav/share/clamav 

sudo cp /usr/local/clamav/db/\* /usr/local/clamav/share/clamav/

# Restart services

sudo systemctl daemon-reload 

sudo systemctl restart clamav-daemon clamav-freshclam 

systemctl restart amavisd

## Check Logs

tail -f /usr/local/clamav/logs/clamd.log 

tail -f /usr/local/clamav/logs/freshclam.log

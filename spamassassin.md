# SpamAssassin Configuration on Red OS

This guide configures SpamAssassin for spam filtering, integrated with iRedMail.

## Enable and Verify SpamAssassin Services

# Check the status of SpamAssassin and Amavisd services

systemctl status spamassassin amavisd

# Enable and start services if not running

systemctl enable --now spamassassin amavisd

## Train SpamAssassin

# Create directories for spam and ham training

mkdir -p /var/vmail/train/{spam,ham} 

chown -R amavis:amavis /var/vmail/train

# Update SpamAssassin rules

sa-update

# Train SpamAssassin with spam and ham emails

sa-learn --spam /var/vmail/train/spam 

sa-learn --ham /var/vmail/train/ham

# Set up daily automatic training via cron

echo "0 3 \* \* \* sa-learn --spam /var/vmail/vmail1/irwol.ru/*/Maildir/.Spam/{cur,new} && sa-learn --ham /var/vmail/vmail1/irwol.ru/*/Maildir/.Ham/{cur,new}" &gt;&gt; /etc/crontab

## Configure Amavisd

# Ensure the following lines are present in /etc/amavisd/amavisd.conf

$sa_tag_level_deflt = -999; # Always add SpamAssassin headers 

$log_level = 5;

@policy_bank{'MYNETS'} = { 

    \[///\], 

    spam_checks =&gt; 1, # Enable spam checks even for internal emails 

};

# If SELinux is enabled, allow Amavis access to SpamAssassin

semanage permissive -a amavis_t

# Update rules and restart services

sa-update && systemctl restart amavisd spamassassin

## Test Spam Filtering

# Create a test user in iRedAdmin (http://localhost/iredadmin)

# Note: Password must include a special character

# Send a test spam email

echo "XJS*C4JDBQADN1.NSBN3*2IDNEN*GTUBE-STANDARD-ANTI-UBE-TEST-EMAIL*C.34X" | mail -s "Spam test" user@irwol.ru

# Log in as the test user and check the Spam folder for the email

# Verify spam detection in logs

tail /var/log/maillog -n 50 | grep 'Spam'

# A "Hits: 1000" in the logs indicates the message was identified as absolute spam

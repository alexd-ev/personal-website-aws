# Personal Portfolio Website With AWS Deployment
## Overview
My personal portfolio website project found here: https://github.com/alexd-ev/personal-website now showcases backend deployment on Amazon EC2.

## Technical Specification
For backend deployment using the AWS Management Console is preferred, but can be achieved using the AWS CLI.

## Amazon EC2
Launch an EC2 instance. This example uses an Ubuntu Server 26.04 (64-bit) VM image, and the t3.micro instance type is sufficient for this application. Create a login key pair, then use these inbound security group rules:
- `SSH` for your IP, to SSH only from your current machine for security
- `Custom TCP`: port 7001 for anywhere, so any machine can connect to the website through the application port
- `HTTP` for anywhere, so any machine can connect to the website through HTTP
- `HTTPS` for anywhere, so any machine can connect to the website through HTTPS
- `All ICMP - IPv4` for your IP, to diagnose instance connectivity issues from your current machine

Set the private key permissions to read only for owners only, then copy the Public IPv4 DNS address of the running instance and SSH into it by running this command in a terminal:
```bash
ssh -i path/to/private_key_file/name.pem ubuntu@<public_dns_address>
```

Once inside the instance, update the package list and upgrade core linux components:
```bash
sudo apt update
sudo apt upgrade -y
```
Install JDK 25 which includes the JRE to run the application (by default the Ubuntu 26.04 repositories installs this version):
```bash
sudo apt install default-jdk -y
```
Test the Javalin web server by first downloading the released `alexds-personal-website-1.0.0.zip` archive found here: https://github.com/alexd-ev/personal-website/releases/latest. Then copy the archive from your host machine to the instance's default home directory using SCP:
```bash
scp -i path/to/private_key_file/name.pem alexds-personal-website-1.0.0.zip ubuntu@<public_dns_address>:
```
Back to the instance, extract the files using `unzip`:
```bash
sudo apt install unzip
unzip alexds-personal-website-1.0.0.zip
```
Run the application using Java:
```bash
java --enable-native-access=ALL-UNNAMED -jar alexds-personal-website-1.0.0.jar
```
Go to http://instance-public-dns-address:7001/

The next step is to use the Apache HTTP Server for this example, to forward HTTP requests to the Javalin web server (reverse proxy). Install it using the `apt` package manager:
```bash
sudo apt install apache2 -y
```
Enable reverse proxy features:
```bash
sudo a2enmod proxy_http
```
Create the website's configuration file:
```bash
sudo vim /etc/apache2/sites-available/personal-website.conf
```
Then paste the following which adds the proxy rule:
```
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://localhost:7001/
    ProxyPassReverse / http://localhost:7001/

    ErrorLog ${APACHE_LOG_DIR}/personal-website-error.log
    CustomLog ${APACHE_LOG_DIR}/personal-website-access.log combined
</VirtualHost>
```
Disable default Ubuntu welcome page and enable the website's main page:
```bash
sudo a2dissite 000-default.conf
sudo a2ensite personal-website.conf
sudo systemctl restart apache2
```
The next step is to make the Java process persistent, so it will constantly run in the background even if the instance restarts. Create a service file:
```bash
sudo vim /etc/systemd/system/personal-website.service
```
Then paste the following which configures the systemd service for the website:
```
[Unit]
Description=Personal Portfolio Website
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu
ExecStart=/usr/bin/java --enable-native-access=ALL-UNNAMED -jar /home/ubuntu/alexds-personal-website-1.0.0.jar
SuccessExitStatus=143
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```
Enable the created service and start it:
```bash
sudo systemctl enable personal-website
sudo systemctl start personal-website
```
Go to http://instance-public-dns-address/

View application logs:
```bash
sudo journalctl -u personal-website -f
```
If a contact message has been submitted, can view via the `sqlite3` CLI:
```bash
sudo apt install sqlite3
sqlite3 database/AboutMe.db
SELECT * FROM contact_messages;
```

# Author
`alexd-ev` (Alex Davidson)

Copyright Alex Davidson (c) 2026

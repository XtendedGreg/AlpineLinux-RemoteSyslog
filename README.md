# AlpineLinux-RemoteSyslog
Remote Syslog with TLS Encryption as seen on XtendedGreg YouTube Channel Stream: https://youtube.com/live/Nh8Up-d_8go

# Basic Setup
## Server
- Install packages ```apk add rsyslog util-linux```
- Edit the rsyslog Configuration File ```vi /etc/rsyslog.conf```
 -- Add the following to the bottom of /etc/rsyslog.conf in insert mode (press "i" and an I should show in the bottom left corner)
```
module(load="imudp")
input(type="imudp" port="514")
```
 - Write changes to file and exit vim (press "esc" key, then type ```:wq``` and press "enter")
- Restart rsyslog service ```rc-service rsyslog restart```
- Add rsyslog to start on boot ```rc-update add rsyslog default```
- Test Server
 - Run the following command ```logger -n 127.0.0.1 -P 514 -T -d "Test Message"```
 - Check /var/log/messages for the test message ```tail /var/log/messages```
- Save configuration with LBU ```lbu commit -d```

## Client
- Install packages ```apk add rsyslog util-linux```
- Edit the rsyslog Configuration File ```vi /etc/rsyslog.conf```
 - Add the following to the bottom of /etc/rsyslog.conf in insert mode (press "i" and an I should show in the bottom left corner)
```
*.* @[your-syslog-server-IP]:514
```
 - Write changes to file and exit vim (press "esc" key, then type ```:wq``` and press "enter")
- Restart rsyslog service ```rc service rsyslog restart```
- Add rsyslog to start on boot ```rc-update add rsyslog default```
- Test Client
 - Run the following command ```logger "Test log from client"```
 - Check /var/log/messages on the SERVER for the test message ```tail /var/log/messages```
- Save configuration with LBU ```lbu commit -d```

# TLS Setup
## Server
- Install packages ```apk add rsyslog util-linux rsyslog-tls openssl```
- Generate CA Certificate
 - Generate CA Private Key ```openssl genrsa -out ca.key 4096```
 - Create CA Certificate using Private Key ```openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 -out ca.pem```
- Create Server Private Key ```openssl genrsa -out server.key 4096```
- Create Server Certificate Signing Request (CSR) using Server Key ```openssl req -new -key server.key -out server.csr```
- Sign the CSR with the CA Key ```openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256```
- Place CA Certificate, Server Certificate and Server Private Key in /etc/rsyslog.d/certs
```
mkdir -p /etc/rsyslog.d/certs
cp ca.pem /etc/rsyslog.d/certs
cp server.crt /etc/rsyslog.d/certs
cp server.key /etc/rsyslog.d/certs
```
- Edit the rsyslog Configuration File ```vi /etc/rsyslog.conf```
 - Add the following to the bottom of /etc/rsyslog.conf in insert mode (press "i" and an I should show in the bottom left corner)
```
module(load="imtcp" StreamDriver.Name="gtls" StreamDriver.Mode="1" StreamDriver.Authmode="anon")
input(type="imtcp" port="6514")

# Specify paths to your certificates
global(DefaultNetstreamDriver="gtls")
global(DefaultNetstreamDriverCAFile="/etc/rsyslog.d/certs/ca.pem")
global(DefaultNetstreamDriverCertFile="/etc/rsyslog.d/certs/server.crt")
global(DefaultNetstreamDriverKeyFile="/etc/rsyslog.d/certs/server.key")
```
 - Write changes to file and exit vim (press "esc" key, then type ```:wq``` and press "enter")
- Restart rsyslog service ```rc-service rsyslog restart```
- Add rsyslog to start on boot ```rc-update add rsyslog default```
- Save configuration with LBU ```lbu commit -d```

## Client
- Install packages ```apk add rsyslog util-linux rsyslog-tls openssl```
- Generate Client Private Key ```openssl genrsa -out client.key 4096```
- Generate Client CSR ```openssl req -new -key client.key -out client.csr```
 - ```cat client.csr``` and copy to SERVER as client.crt ```echo "[paste copied contents of client.csr here]" > client.csr```
- On SERVER
 - Sign the client CSR with the CA Key ```openssl x509 -req -in client.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256```
 - ```cat client.crt``` and copy to CLIENT as client.crt ```echo "[paste copied contents of client.crt here]" > client.crt```
 - ```cat ca.pem``` and copy to client as ca.pem ```echo "[paste copied contents of ca.pem here]" > ca.pem```
- Place CA Certificate, Client Certificate, and Client Private Key in /etc/rsyslog.d/certs
```
mkdir -p /etc/rsyslog.d/certs
cp ca.pem /etc/rsyslog.d/certs
cp client.crt /etc/rsyslog.d/certs
cp client.key /etc/rsyslog.d/certs
```
- Edit the rsyslog Configuration File ```vi /etc/rsyslog.conf```
 - Add the following to the bottom of /etc/rsyslog.conf in insert mode (press "i" and an I should show in the bottom left corner)
```
# certificate files - just CA for a client
global(DefaultNetstreamDriverCAFile="/etc/rsyslog.d/certs/ca.pem")
global(DefaultNetstreamDriverCertFile="/etc/rsyslog.d/certs/client.crt")
global(DefaultNetstreamDriverKeyFile="/etc/rsyslog.d/certs/client.key")

# set up the action for all messages
action(type="omfwd" protocol="tcp" target="[your-syslog-server-IP]" port="6514" StreamDriver="gtls" StreamDriverMode="1" StreamDriverAuthMode="anon")
```
 - Write changes to file and exit vim (press "esc" key, then type ```:wq``` and press "enter")
- Restart rsyslog service ```rc-service rsyslog restart```
- Add rsyslog to start on boot ```rc-update add rsyslog default```
- Test Client
 - Run the following command ```logger "Test log from client"```
 - Check /var/log/messages on the SERVER for the test message ```tail /var/log/messages```
- Save configuration with LBU ```lbu commit -d```

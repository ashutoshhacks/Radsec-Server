# Radsec-Server
This project documents the setup and configuration of RadSec (RADIUS over TLS) using FreeRADIUS on a Linux server and a MikroTik router as the client (you can use any client. The server configuration stays the same). It covers certificate management, server configuration, and secure authentication to ensure encrypted RADIUS communication.

# Prerequisites
A Linux server (Ubuntu/Debian recommended)

FreeRADIUS installed on the server

MikroTik router as the RADIUS client

OpenSSL for certificate generation

# Install FreeRADIUS
```sudo apt update```

```sudo apt upgrade -y```

```sudo apt install php apache2 php8.1-fpm freeradius libapache2-mod-php mariadb-server freeradius-mysql freeradius-utils php-{gd,common,mail,mail-mime,mysql,pear,db,mbstring,xml,curl} -y```

```sudo systemctl enable --now apache2 && sudo systemctl enable freeradius```

# Generate Certificates and Keys
I am using a Linux machine as my root CA and will be generating all the necessary certificates and keys on this machine. You can install openSSL from [here](https://docs.oracle.com/en/cloud/paas/integration-cloud/ftp-adapter/install-and-configure-openssl.html) if not already installed.

### Create a private CA
```openssl req -x509 -newkey rsa:2048 -keyout rootCA-key.pem -out rootCA-cert.pem -days 365 -nodes```

### Generate Server Certificate and sign using private CA
**TIP: If you do not have a fully qualified domain name in your network, you can configure the CN (Common Name) of the server and client certificates as their IP addresses and make sure to let the radius client (if possible) also know to allow 'FQDN allow ip address'. Root CA CN can be anything.**

```openssl req -x509 -newkey rsa:2048 -keyout server-key.pem -out server-cert.pem -days 365 -nodes```

```openssl req -new -newkey rsa:2048 -keyout server-key.pem -out server.csr -nodes```

```openssl x509 -req -in server.csr -CA rootCA-cert.pem -CAkey rootCA-key.pem -CAcreateserial -out server-cert.pem -days 365```

### Generate Client Certificate and sign using private CA
```openssl req -x509 -newkey rsa:2048 -keyout client-key.pem -out client-cert.pem -days 365 -nodes```

```openssl req -new -newkey rsa:2048 -keyout client-key.pem -out client.csr -nodes```

```openssl x509 -req -in client.csr -CA rootCA-cert.pem -CAkey rootCA-key.pem -CAcreateserial -out client-cert.pem -days 365```

# Radsec Server Configuration
```sudo -i```

```service freeradius stop```

```cd /etc/freeradius/3.0/sites-enabled```

```ln -s ../sites-available/tls```

```vim /etc/freeradius/3.0/sites-available/tls```


**Find 'clients radsec {' and configure the client details or it should look like the below after editing**

```
clients radsec {
       client mikrotik {
               ipaddr = 192.168.10.1
               proto = tls
               virtual_server = default
       }
```
**Don't forget to copy your certificate files and keys which you just created in your Linux machine to your server. After copying the required CA, certificate and key files in your server change the default path in the tls file.**

```

```

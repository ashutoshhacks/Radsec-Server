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

**NOTE: Make sure the secret should be set to radsec in both server and client.**

```
clients radsec {
       client mikrotik {
               ipaddr = 192.168.10.1
               proto = tls
               secret = radsec
       }
```
**Don't forget to copy your certificate files and keys which you just created in your Linux machine to your server. After copying the required CA, certificate and key files in your server, change the default path in the tls file.**

```
tls{
    private_key_file = /etc/ssl/private/server2-key.pem
    .
    .
    certificate_file = /etc/ssl/certs/server2-cert.pem
    .
    .
    ca_file = /etc/ssl/certs/ca-cert.pem
}
```

### Edit the /etc/freeradius/3.0/sites-enabled/default file so that the beginning of the authorize and preacct sections looks as follows:
```
authorize {
    accept
    ...
}
...
preacct {
    handled
    ...
}
```
### Edit /etc/freeradius/3.0/clients.conf to look like below
```
client mikrotik {
        ipaddr = 192.168.10.1
        secret = radsec
        require_client_certificate = yes
        ca_file = /etc/ssl/certs/ca-cert.pem
}
```

### Edit /etc/freeradius/3.0/users to look like below
```
tester Cleartext-Password := "tester123"
# all users here
```
### Start freeradius in debug mode 
```freeradius -X OR freeradius -fxxl /dev/stdout```

# Radsec Client Configuration
I have used a mikrotik router as a radius client in my case and the configuration is simple and straightforward. 

**Step 1-** Copy the client certificate, key and CA into mikrotik files.

**Step 2-** Go into Certificates > Import > import the client certificate, key, and CA file.

**Step 3-** Go into RADIUS and configure address of your radsec server, protocol to radsec, secret to "radsec", authentication and accounting ports to 2083, and lastly select the client certificate + key pair you just added.

# Troubleshooting
Always restart the freeradius service after making changes in the configuration files and stop the service to into debug mode.

PRO TIP- Sometimes the freeradius service would not restart due to several issues. At this time, just restart the machine to automatically start the freeradius after bootup.
# Conclusion
This setup provides a secure RADIUS authentication mechanism over TLS. RadSec ensures encrypted communication, improving security over traditional RADIUS setups.

### If this project helped you, give it a ⭐ and contribute by submitting issues or pull requests!

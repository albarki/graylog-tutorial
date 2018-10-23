# How to ship logs to Graylog2.1  using collector-sidecar and Filebeat

### Introduction 
The **Filebeat** is Lightweight shipper used to forward logs and files from remote client servers to Centralized logging server like `Graylog` or `Logstash`.

The **Graylog Collector Sidecar** is a supervisor process for 3rd party log collectors like Filebeat . it allows us to manage Filebeat from one central place `Graylog Web interface` instead of logging to each remote server and change the configuration from it.

## Our Goal
The goal of this tutorial is to configure the sidecar using Graylog Web interface to collect Apache logfiles and ship them with a Filebeat collector to an input listening on port 5044 on Graylog Server.

Our Setup has the following components:
- Graylog 2.1 server up and running

- CentOS server: we need to ship Apache logs from here.


![Setup structure](http://storage5.static.itmages.com/i/16/1026/h_1477484686_9510569_7f6432b259.png)



## Prerequisites 

Before you begin this guide you'll need the following:

- GrayLog 2.1 server up and running.

- CentOS Server has Apache installed on it.
 

## Install Filebeat and Sidecar

The Filebeat binaries are included in the Sidecar package so we only need to install one package.

<$>[note]
**Note:** The following commands run on all client servers where you want to ship logs from it, in our case we will run these commands on the CentOS server which has Apache on it.
<$>

Download pre-compiled package from [the Github release page of the project](https://github.com/Graylog2/collector-sidecar/releases) :
```super_user
wget https://github.com/Graylog2/collector-sidecar/releases/download/0.1.0-alpha.2/collector-sidecar-0.1.0-1.x86_64.rpm
```
Install the RPM package :
```super_user
yum localinstall collector-sidecar-0.1.0-1.x86_64.rpm 
```
Open the configuration file of collector-sidecar with your favorite editor :
```super_user
vim /etc/graylog/collector-sidecar/collector_sidecar.yml
```
Delete the file's contents, and paste the following code block into the file. Be sure to update `server_url` to match the IP of your Graylog server also you can change `list_log_files` to which files you want to ship to the Graylog Server :


```collector-sidecar.yml
[label /etc/graylog/collector-sidecar/collector_sidecar.yml]
server_url: http://<^><Graylog-server-ip><^>:9000/api/
update_interval: 10
tls_skip_verify: false
send_status: true
list_log_files:
  - <^>/var/log<^>
  - <^>/etc/apache2/logs<^>
collector_id: file:/etc/graylog/collector-sidecar/collector-id
log_path: /var/log/graylog/collector-sidecar
log_rotation_time: 86400
log_max_age: 604800
tags:
    - linux
    - apache
sidecar/generated/nxlog.conf
    - name: filebeat
      enabled: true
      binary_path: /usr/bin/filebeat
      configuration_path: /etc/graylog/collector-sidecar/generated/filebeat.yml

}
```
Save and exit, then Start the Sidecar service:
```super_user
 graylog-collector-sidecar -service install
 systemctl start collector-sidecar
```

##  Generate SSL certificate

Since we are going to use Filebeat to ship logs from our Client Servers to our Graylog server, we need to create an SSL certificate and key pair to make sure that the communication from Filebeat and Graylog is encrypted.

<$>[note]
**Note:** The following commands run on Graylog Server.
<$>

add Graylog IP address to the `subjectAltName`. To do so, open the Openssl Configuration file:
```super_user
vim /etc/ssl/openssl.cnf
```
Find the `[ v3_ca ]` in the file, and add under it the following:
```openssl.cnf
[label /etc/ssl/openssl.cnf]
[ v3_ca ]
subjectAltName = IP:<^><Graylog-IP-address><^>
```
Save and Exit.

Now let's generate the certificate and the private key:
```super_user
mkdir -p /etc/pki/tls/certs
mkdir -p /etc/pki/tls/private
cd /etc/pki/tls
openssl req -config /etc/ssl/openssl.cnf -x509 -days $((100 * 365)) -batch -nodes -newkey rsa:2048 -keyout private/graylog.key -out certs/graylog.crt
```
Graylog prefere the key in the format of PKCS#8, so run this command to convert it :
```super_user
cd /etc/pki/tls/private
openssl pkcs8 -nocrypt -topk8 -in graylog.key -out graylog.pk8
```
##  Copy SSL Certificate

Copy the SSL Certificate we just created in the previous step to your Client Server


```custom_prefix(grylog$)
scp /etc/pki/tls/certs/graylog.crt <^>user<^>@<^>client_server_ip_address<^>:/tmp
```
Next access your Client Server and copy the SSL certifcate to the proper place:

```custom_prefix(Client_server$)
mkdir -p /et/pki/tls/certs
cp /tmp/graylog.crt /etc/pki/tls/certs
```


##  Configure Graylog Server

Create a Beats input where collectors can send data to, Acess the Web interface of Graylog 2.1, then go to `System-> Inputs`

![create Input](http://pix.toile-libre.org/upload/original/1477490876.png)

Configure the input as follows:

![configure Input](http://pix.toile-libre.org/upload/original/1477494122.png)

- **Title**: pick your favourite name ex: `beats-input` 

- check right **Global**

- **Bind address** : `0.0.0.0`

- **Port**: 5044

- **TLS cert file (optional)**: `/etc/pki/tls/certs/graylog.crt`

- **TLS private key file (optional)**: `/etc/pki/tls/private/graylog.pk8`

- **TLS Client Auth Trusted Certs(optional)**: `/etc/pki/tls/certs/graylog.crt`

- Check right **Enable TLS (optional)**
 
Make sure that input configuration is right, then press save.


Now navigate to the collector configurations. click on `System -> Collectors -> Manage configuration` in your Graylog web interface

![access collectors](http://storage2.static.itmages.com/i/16/1026/h_1477487662_2853681_f3182d5333.png)

![access collectors](http://pix.toile-libre.org/upload/original/1477488058.png)

Next, create a new configuration:
![create configuration](http://pix.toile-libre.org/upload/original/1477494957.png)

Next, Choose a name for the configuration and click save button:
![name configuration](http://pix.toile-libre.org/upload/original/1477495092.png)

Next, Click on the new configuration and create Filebeat output.
![create filebeat output](http://pix.toile-libre.org/upload/original/1477495355.png)

Next, Configure the output as follows:
- **Name** : pick any name ex:`apache-output`

- **Type**: `[FileBeat] Beats output`

- **Hosts** : `['<^><Graylog-IP-address><^>:5044']`

- check right **Enable TLS support**

- **CA File** : `/etc/pki/tls/certs/graylog.crt`

Make sure every thing is OK, then click save.
![configure filebeat output](http://pix.toile-libre.org/upload/original/1477495832.png)

Next, Create a Filebeat file input to collect the Apache access logs:

![create filebeat input](http://pix.toile-libre.org/upload/original/1477496012.png)

Next, Configure the input as follows:

- **Name** : pick any name ex:`apache-input`

- **Forward to (Required)**: The name of the output we just created `apache-output`

- **Type** : `[FileBeat] file input`

- **Path to Logfile** : Location of the log file you want to ship from the Client Server to Graylog ex:`/etc/apache2/logs/*.log`


![configure filebeat input](http://pix.toile-libre.org/upload/original/1477496279.png)

When you finish click `save`

The last step Tag the configuration with the `apache` tag. write the tag name in the field then press `Update tags`.

![update tags](http://pix.toile-libre.org/upload/original/1477496800.png)



## Conclusion

Now it's easily using Filebeat and Sidecar to manage the configuration of Client Servers from the Graylog Web interface. 

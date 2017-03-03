elk_docker
==========

ELK (Elasticsearch, Logstash, Kibana) as Docker Container.
You can use it for demos or as a development environment e.g.

Version
==========

elasticsearch 2.1

logstash 2.1

kibana 4.3

This setup allows for watching files in a local directory and log event forwarding over a TLS-secured connection.

## Automatic Container Setup

This repository contains a Vagrantfile which allows for automated provisioning of a virtual machine and running all defined docker containers inside it.

Just three steps are required:

1. Install [Vagrant](https://www.vagrantup.com/)
2. Run `vagrant up` in the directory containing the Vagrantfile
3. Access http://localhost:5601 in your web browser to open kibana

To monitor and restart docker containers enter the virtual machine via `vagrant ssh`.

## Manual Container Setup

First of all, build images from all the Dockerfiles. Therefore, change dir into the subfolders and run

```sh
  $ cd elasticsearch
  $ sudo docker build -t elasticsearch .
  $ cd ../logstash
  $ sudo docker build -t logstash .
  $ cd ../kibana
  $ sudo docker build -t kibana .
```

Then start a container for Elasticsearch, Logstash, and Kibana, respectively.

```sh
  $ cd ..
  $ sudo docker run --name elasticsearch -d -p=9200:9200 elasticsearch
  $ mkdir logs
  $ sudo docker run --name logstash -d -p=5000:5000 --link elasticsearch:elasticsearch \
      -v `pwd`/logstash/config:/conf -v `pwd`/logs:/var/logstash/logs logstash
  $ sudo docker run --name kibana -d -p=5601:5601 --link elasticsearch:elasticsearch kibana
```

Finally, go to http://localhost:5601 to access kibana.

**Container Parameter Details:**

* all containers are named appropriately (`--name`) and run in the background (`-d`).
* elasticsearch exposes its HTTP service port 9200.
* logstash exposes port 5000 for (TLS-secured) log forwarding.
* kibana is accessible on port 5601.
* the logstash configuration is not embedded in the docker image but mounted as a data volume under '/conf' when the container is started. Thus, the logstash configuration can be updated easily and a restart of the logstash container is sufficient to apply the changes (`docker restart logstash`).
* a local log file directory 'logs/' is created and mounted in the logstash container. It is watched for any added or updated log files. 

**Container Inspection:**

Verify that all containers are running.

```sh
  $ sudo docker ps -a
```

Check for any errors in the logstash container log.

```sh
  $ sudo docker logs logstash
```

## Adding Log Messages

The logstash configuration (logstash/logstash.conf) defines two input options for processing log events:

* via log files in the 'logs/' directory.
* via TLS-secured log forwarding on port 5000.

### Watching Log Files

The easiest options for adding log messages is to

* copy a line-based log file to the local 'logs/' directory, e.g.  
  `sudo cp /var/log/dmesg logs/`
* append lines of log messages to a log file in the local 'logs/' directory, e.g.  
  ```
  sudo chown `whoami` logs && echo "Hello World" >>logs/test.log
  ```

### Forwarding Log Events Via TLS

First of all, the security certificate needs to be obtained from the logstash docker image.

```sh
sudo docker cp logstash:/etc/pki/tls/certs/logstash-ca.crt .
```

OpenSSL can be used for testing. Type following command to open the connection and then enter any lines of log messages.

```sh
openssl s_client -quiet -CAfile logstash-ca.crt -connect localhost:5000
```

[NXlog](http://nxlog.co/products/nxlog-community-edition) is a full-featured log processor which can also forward logs via TLS. A basic NXlog setup is provided in the nxlog container.

```sh
$ cp logstash-ca.crt nxlog/
$ cd nxlog
$ sudo docker build -t nxlog .
$ cd ..
$ sudo docker run --name nxlog -d --link logstash:logstash -v `pwd`/nxlog/config:/conf nxlog
```

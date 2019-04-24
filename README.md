# elk-stack
This project deploy a stack composed by Elasticsearch, Kibana and Logstash plus RabbitMQ and a demo app connected to the ELK stack through Rabbit. The work is based on [Piotr Mińkowski project](https://piotrminkowski.wordpress.com/2017/02/03/how-to-ship-logs-with-logstash-elasticsearch-and-rabbitmq/) and improves it adding:
* A deploy using docker-compose instead of multiple docker commands avoiding usage of deprecated --link option
* An improved logstash configuration wich auto-configures RabbitMQ and Logstash
* A tutorial to deploy the project on Google Cloud Platform using container-optimized machine
* ELK stack update to version 6.7.1 (release date April 2, 2019). Version 7.0.0 has been release on April 10 2019 but has not been tested yet

## Docker Structure
### Containers
This project uses following docker images:
* elasticsearch: plain elastic search container
* kibana: kibana dashboard, connected to elastic search to view logs
* rabbit: rabbit:management image wich provides RabbitMQ and RabbitMQ admistration panel 
* logstash: logstash image, logstash.conf file is mounted as volume and contains RabbitMQ setup to initialize queue and exchange without manual setup
* web: openJDK 8 image which run `java -jar` command to start web client and mounts requested jar as volume. jar file can be obtained running mvn build

### Networking
To avoid --link deprecated option and reducing the number of exposed ports bridge network `elknet` has been created. Please not that elastic search ports 9200 and 9300 are no longer exposed. 

Exposed ports:
* 5601: Kibana dashboard
* 5672: RabbitMQ port exposed to connect easily a project running outside docker (e.g. launched from IDE) but is not required to run the project inside docker
* 15672: RabbitMQ administration panel(username:`guest` password:`guest`)
* 8080: REST APS


### Setup
Move inside project directory and run 
```
docker-compose up
```

## Google Cloud Configuration
Machine creation and configuration can be found in Google Cloud platform documentation [reference](https://cloud.google.com/community/tutorials/docker-compose-on-container-optimized-os). This tutorial adds some customization.

### Machine Setup
1. Open the Google Cloud Platform console.
2. Create a new Compute Engine instance.
3. Select the desired Zone.
4. Select the desired Machine type.
5. Change the Boot disk to "Container-Optimized OS stable".
6. Click the Create button to create the Compute Engine instance.

### Running Docker Compose

Container optimized machine does not provide docker-compose out of the box. To run docker-compose command you need to use a dedicated container 

Create an alias to run this container using `docker-compose` command.
Please note that `/rootfs` has been removed to allow container reasd and write.

```
echo alias docker-compose="'"'docker run --rm \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v "$PWD:/$PWD" \
    -w="/$PWD" \
    docker/compose:1.24.0'"'" >> ~/.bashrc
```

and reload configuration

```
source ~/.bashrc
```
Increase virtual memory limit to start elastic search

```
sudo sysctl -w vm.max_map_count=262144
```
[reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/vm-max-map-count.html)


### Network configuration
To expose services on the internet create a firewall rule as follow:
Go to cloud.google.com and select project, then
* Select Networking/VPC network
* Select "Firewalls rules"
* Select "Create Firewall Rule"
* Expose ports tcp:5601, tcp:5672, tcp:15672, tcp:8080  to "IP ranges" 0.0.0.0/0
* Create target tags "elk-rabbit"
* Save

Then edit VM instance details and add in "Network tags" section the new tag "elk-rabbit"

[reference](https://cloud.google.com/vpc/docs/using-firewalls)

### ELK Stack Startup
Clone project directory

```
git clone https://github.com/alessandrovalentini/elk-stack.git
```

Move into directory and run the project

```
cd ~/elk-stack
docker-compose up
```

### Demo instance-2
Improved version with log levels
* Kibana dashboard: http://35.228.203.207:5601
* Rabbit management: http://35.228.203.207:15672
* Log API: http://35.228.203.207:8080/log/i/log1

# kd : kafka on docker for testing purpose
Just follow these steps to setup a kafka cluster and kafka-manager running in containers.

## requirements
* docker >= 1.12
* docker-compose >= 1.8 (because of version 2 syntax in yaml compose file)
* user needs to be in the _docker_ system group (or to be _root_)

## clone repository
```
git clone git@github.com:ledroide/kd.git
```
## run a Kafka cluster including multiple nodes + zookeeper + kafka-manager
* pwd near the compose file
```
cd kd
```
* check and pull the docker images :
```
docker-compose build --pull
```
* launch instances :
```
docker-compose up -d 
```
* scale up to 4 kafka nodes (until the compose file does not support any _scale_ directive) :
```
export KAFKANODES=4
docker-compose scale kafka=${KAFKANODES}
```
## create some random topics
* install pwgen (here with ubuntu _yakkety yak_ ; if older version use apt-get instead, if redhat use yum) :
```
[ -x /usr/bin/pwgen ] && echo "Already installed" || ( apt update && apt install pwgen )
```
 > we hijack the password generator for creating topic names.

* create random topics with some random parameters :
```
NAMELENGHT=10
export NBTOPIC=100
for TopicID in $(/usr/bin/pwgen -AB10 ${NAMELENGHT} ${NBTOPIC}) 
  do docker-compose run kafka /opt/kafka/bin/kafka-topics.sh --create \
   --zookeeper zookeeper:2181 \
   --replication-factor $((RANDOM%(${KAFKANODES}-1)+2)) \
   --partitions $((RANDOM%(${KAFKANODES}-1)+2)) \
   --topic ${TopicID}
done
```
## use producers and consumers
* store locally the topics list :
```
docker-compose run kafka /opt/kafka/bin/kafka-topics.sh \
 --zookeeper zookeeper:2181 --list > /tmp/alltopics
echo "Number of topics in the cluster : $(wc -l /tmp/alltopics | cut -d ' '  -f 1)"
```
* pick up a topic from the list, store it in a file and in a variable
```
/bin/sed "$((RANDOM%(${NBTOPIC})+1))q;d" /tmp/alltopics  > /tmp/onetopic && \
 export AVGTOPIC=$(/bin/cat /tmp/onetopic) && \
 export CLEANTOPIC="${AVGTOPIC//[$'\t\r\n ']}" ; \
 echo "Current random topic is $CLEANTOPIC"
```
* start a console as an interactive producer to the random topic :
```
docker-compose run --name "pub_${CLEANTOPIC}" --rm kafka \
 kafka-console-producer.sh --broker-list kafka:9092 --topic ${CLEANTOPIC}
```
* and another console with a consumer console :
```
export AVGTOPIC=$(/bin/cat /tmp/onetopic) && \
 export CLEANTOPIC="${AVGTOPIC//[$'\t\r\n ']}" ; \ 
 echo "Current random topic is $CLEANTOPIC"
docker-compose run --name sub_${CLEANTOPIC} --rm kafka \
 kafka-console-consumer.sh --zookeeper zookeeper:2181 \
 --from-beginning --topic ${CLEANTOPIC}
```

## Kafka Manager interface
* open the manager interface in you browser : [http://localhost:9000/addCluster](http://localhost:9000/addCluster)
* add a new cluster
  * Cluster name : _whatever_
  * Cluster Zookeeper hosts : _zookeeper:2181_
  * enable JMX polling
* select the new cluster and play

## performance testing
* tip to generate multiple messages to a broker as a producer, just like a basic _lorem ipsum_ :
```
tr -dc a-z1-4 </dev/urandom | tr 1-2 ' \n' | awk 'length==0 || length>50' | tr 3-4 ' ' | sed 's/^ *//' | cat -s | sed 's/ / /g' | fmt
```

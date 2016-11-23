Clone repository
------------------


Run a Kafka cluster including 4 KAFKANODES + zookeeper + kafka-manager
---------------------------------------------------------------------
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
* scale up to 4 kafka KAFKANODES :
```
export KAFKANODES=4
docker-compose scale kafka=${KAFKANODES}
```

Create some random topics
------------------------
* install pwgen (here with ubuntu _yakkety yak_ ; if older version use apt-get instead, if redhat use yum) :
```
[ -x /usr/bin/pwgen ] || ( apt update && apt install pwgen )
```
Pwgen is a password generator that we hijack here to generate topic names.
* create random topics with some random parameters :
```
NAMELENGHT=10
export NBTOPIC=100
for TopicID in $(/usr/bin/pwgen -AB10 ${NAMELENGHT} ${NBTOPIC}) 
  do docker-compose run kafka /opt/kafka/bin/kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor $((RANDOM%(${KAFKANODES}-1)+2)) --partitions $((RANDOM%(${KAFKANODES}-1)+2)) --topic ${TopicID}
done
```

Use producers and consumers
---------------------------
* store locally the topics list :
```
docker-compose run kafka /opt/kafka/bin/kafka-topics.sh --zookeeper zookeeper:2181 --list > /tmp/alltopics
```
* pick up a topic from the list, store it in a file and in a variable
```
/bin/sed "$((RANDOM%(${NBTOPIC})+1))q;d" /tmp/alltopics  > /tmp/onetopic && export AVGTOPIC=$(/bin/cat /tmp/onetopic) && export CLEANTOPIC="${AVGTOPIC//[$'\t\r\n ']}" ; echo "Current random topic is $CLEANTOPIC"
```
* start a console as an interactive producer to the random topic :
```
docker-compose run --name "pub_${CLEANTOPIC}" --rm kafka kafka-console-producer.sh --broker-list kafka:9092 --topic ${CLEANTOPIC}
```
* and another console with a consumer console :
```
export AVGTOPIC=$(/bin/cat /tmp/onetopic) && export CLEANTOPIC="${AVGTOPIC//[$'\t\r\n ']}" ; echo "Current random topic is $CLEANTOPIC"
docker-compose run --name sub_${CLEANTOPIC} --rm kafka /opt/kafka/bin/kafka-console-consumer.sh --zookeeper zookeeper:2181 --from-beginning --topic ${CLEANTOPIC}
```



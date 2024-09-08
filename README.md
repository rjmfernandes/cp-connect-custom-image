# Creating custom confluent docker images to sort CVEs

- [Creating custom confluent docker images to sort CVEs](#creating-custom-confluent-docker-images-to-sort-cves)
  - [Manual Exploration](#manual-exploration)
  - [Automate](#automate)
  - [Testing](#testing)


## Manual Exploration

Check CVEs of the image:

```
docker scout cves confluentinc/cp-server-connect:7.6.2-3-ubi8
```

Check High vulnerabilities on response. As of day 20240908 we have 4 related to the following CVEs (just listing the ones with fixes):

- HIGH CVE-2019-20916 (packages pkg:pypi/pip@9.0.3)
- HIGH CVE-2024-6345 (packages pkg:pypi/setuptools@39.2.0 pypi/setuptools@67.8.0)
- HIGH CVE-2022-40897 (packages pkg:pypi/setuptools@39.2.0)

Run the container of the image

```
docker run confluentinc/cp-server-connect:7.6.2-3-ubi8  /bin/bash -c "echo 'Hello World'; sleep infinity"
```

List docker container and copy its id:

```
docker ps
```

Run as root an interactive shell on container:

```
docker exec --user root -it 618a7d71c4e8 bash
```

List packages installed:

```
yum list installed|grep pip
yum list installed|grep setuptools
```

We would be interested in the following packages related with CVEs listed before:

- platform-python-pip.noarch            9.0.3-24.el8                            @ubi-8-baseos-rpms
- platform-python-setuptools.noarch     39.2.0-7.el8                            @ubi-8-baseos-rpms


Remove the problematic python packages:

```
pip3 uninstall setuptools
pip3 uninstall pip
rpm -e --nodeps platform-python-pip-9.0.3-24.el8.noarch
rpm -e --nodeps python3-pip-wheel-9.0.3-24.el8.noarch
rpm -e --nodeps python39-pip-20.2.4-9.module+el8.10.0+21329+8d76b841.noarch
rpm -e --nodeps python39-pip-wheel-20.2.4-9.module+el8.10.0+21329+8d76b841.noarch
rpm -e --nodeps platform-python-setuptools-39.2.0-7.el8.noarch
rpm -e --nodeps python3-setuptools-wheel-39.2.0-7.el8.noarch
rpm -e --nodeps python39-setuptools-wheel-50.3.2-5.module+el8.10.0+20345+671a55aa.noarch
```

You may see some errors cause we may be trying to remove files already removed.
Please bear in mind that this might break the dnf/yum as they are dependent on the packages being removed. 

Exit and save the image:

```
docker commit 618a7d71c4e8 testimage
```

You can kill the process now:

```
docker kill 618a7d71c4e8
```

Let's check with scout now:

```
docker scout cves testimage
```

You should have no HIGH vulneravilities anymore.

## Automate

Let's run inside the image folder the docker build:

```
cd image
docker build -t customconnect:7.6.2 .
cd ..
```

We can now check our image again just in case:

```
docker scout cves customconnect:7.6.2
```

## Testing 

Let's test our image works fine by running:

```
docker compose up -d
```

Check everything is running fine:

```
docker compose logs -f connect
```

Then let's install `kafka-connect-datagen` connector plugin for populating our topics with sample data:

```shell
docker compose exec connect bash
```

And then inside the container bash:

```shell
confluent-hub install confluentinc/kafka-connect-datagen:latest
```

And we exit the container bash and restart connect:

```shell
docker compose restart connect
```

Again we monitor the Kafka Connect logs and confirm it has started.

After we can create the connectors:

```shell
curl -i -X PUT -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/my-datagen-source1/config -d '{
    "name" : "my-datagen-source1",
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic" : "products",
    "output.data.format" : "AVRO",
    "quickstart" : "SHOES",
    "tasks.max" : "1"
}'
```

Let's check our topic is being populated:

```
kafka-avro-console-consumer --topic products \
--bootstrap-server 127.0.0.1:9092 \
--property schema.registry.url=http://127.0.0.1:8081 \
--property print.key=true \
--property print.value=true \
--value-deserializer io.confluent.kafka.serializers.KafkaAvroDeserializer \
--key-deserializer org.apache.kafka.common.serialization.StringDeserializer \
--from-beginning
```
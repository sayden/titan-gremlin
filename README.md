# Docker Image for Titan Graph Database

<p align="center"><img src="https://raw.githubusercontent.com/elubow/titan-gremlin/master/titan-docker-logo.png" width="250"></p>

[Titan](http://titandb.io/) is a free, open source database that is capable of processing
extremely large graphs and it supports a variety of indexing and storage backends,
which makes it easier to extend than some popular NoSQL Graph databases.

This docker image instantiaties a Titan graph database that is capable of
integrating with an ElasticSearch container (Indexing) and a Cassandra container (Storage).

The default distribution of Titan runs on a single node, so I thought it would be helpful
if there was a modular way at runtime to hook up Titan to its dependencies.

Enter Docker. Now it is possible to run Titan and it's dependencies in separate Docker containers.

## Titan

This container is using Titan 1.0.0. Please refer to
its [page](https://github.com/thinkaurelius/titan/wiki/Downloads) for more information.

## TinkerPop and Gremlin

[TinkerPop](http://www.tinkerpop.com/) is a vendor-independent API specification for
manipulating and access Graph databases. This is using TinkerPop 3.0.1.

## Running

The minimum system requirements for this stack is 1 GB with 2 cores.

```
docker run -d --name es1 elasticsearch:1.5
docker run -d --name cas1 elubow/cassandra
docker run -d -P --name titan1 --link es1:elasticsearch --link cas1:cassandra elubow/titan-gremlin
```

I run with a 3 node Cassandra cluster and some local ports exported, like so:

```
docker run -d --name cas1 -p 7000:7000 -p 7001:7001 -p 7199:7199 -p 9160:9160 -p 9042:9042 elubow/cassandra
docker run -d --name cas2 --link cas1:cassandra elubow/cassandra start docker inspect --format '{{ .NetworkSettings.IPAddress }}' cas1
docker run -d --name cas3 --link cas1:cassandra elubow/cassandra start docker inspect --format '{{ .NetworkSettings.IPAddress }}' cas1

docker run -d --name es1 --link cas1:cassandra -p 9200:9200 elasticsearch:1.5

docker run -d --name titan1 --link es1:elasticsearch --link cas1:cassandra -p 8182:8182 -p 8184:8184 elubow/titan-gremlin
```

## Connecting with Gremlin Client

If you want to connect from a Gremlin client, download [Titan](http://s3.thinkaurelius.com/downloads/titan/titan-1.0.0-hadoop1.zip).
Then create a properties file that looks like this where the `storage.hostname` is the hostname or IP of docker.

```
storage.backend=cassandrathrift
storage.hostname=192.168.99.100
index.search.hostname=192.168.99.100
index.search.elasticsearch.client-only=true
```

Then start the gremlin server by doing `bin/gremlin.sh` and run the following commands inside the Gremlin console:

```
gremlin> graph = TitanFactory.open('/Users/elubow/tmp/local-gremlin.properties')
==>standardtitangraph[cassandrathrift:[192.168.99.100]]
gremlin> g = graph.traversal()
==>graphtraversalsource[standardtitangraph[cassandrathrift:[192.168.99.100]], standard]
gremlin> g.V()
==>v[4168]
```

### Ports

8182: HTTP port for REST API
8184: JMX Port (You won't need to use this, probably)

To test out the REST API (over Boot2docker):

```
curl "http://192.168.99.100:8182?gremlin=100-1"
curl "http://192.168.99.100:8182?gremlin=g.addV('Name','Eric')"
curl "http://192.168.99.100:8182?gremlin=g.V()"
```

## Dependencies

I've tested this container with the following containers:

	- elubow/cassandra: This is the Cassandra Storage backend for Titan. It scales well for large datasets. Also forces Cassandra 2.1 as that's compatible with Titan.
	- elasticsearch v1.5: This is the ElasticSearch Indexing backend for Titan. It provides search capabilities for Titan graph datasets.

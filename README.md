# helidon-erp-api

Helidon SE application that uses a database with JPA

## Build and run


This example requires a database, see [Database Setup](#database-setup).


With JDK 27
```bash
mvn package
java -jar target/helidon-erp-api.jar
```

## Exercise the application

Basic:
```
curl -X GET http://localhost:8080/simple-greet
Hello World!
```


JSON:
```
curl -X GET http://localhost:8080/greet
{"message":"Hello World!"}

curl -X GET http://localhost:8080/greet/Joe
{"message":"Hello Joe!"}

curl -X PUT -H "Content-Type: application/json" -d '{"greeting" : "Hola"}' http://localhost:8080/greet/greeting

curl -X GET http://localhost:8080/greet/Jose
{"message":"Hola Jose!"}
```


Database:
```
# List all Pokémon
curl http://localhost:8080/db/pokemon

# List all Pokémon types
curl http://localhost:8080/db/type

# Get a single Pokémon by id
curl http://localhost:8080/db/pokemon/2

# Get a single Pokémon by name
curl http://localhost:8080/db/pokemon/name/Squirtle

# Add a new Pokémon Rattata
curl -i -X POST -d '{"id":7,"name":"Rattata","idType":1}' http://localhost:8080/db/pokemon

# Rename Pokémon with id 7 to Raticate
curl -i -X PUT -d '{"id":7,"name":"Raticate","idType":2}' http://localhost:8080/db/pokemon

# Delete Pokémon with id 7
curl -i -X DELETE http://localhost:8080/db/pokemon/7
```



## Try metrics

```
# Prometheus Format
curl -s -X GET http://localhost:8080/observe/metrics
# TYPE base:gc_g1_young_generation_count gauge
. . .

# JSON Format
curl -H 'Accept: application/json' -X GET http://localhost:8080/observe/metrics
{"base":...
. . .
```


## Try health

This example shows the basics of using Helidon SE Health. It uses the
set of built-in health checks that Helidon provides plus defines a
custom health check.

Note the port number reported by the application.

Probe the health endpoints:

```bash
curl -X GET http://localhost:8080/observe/health
curl -X GET http://localhost:8080/observe/health/ready
```



### Database Setup

In the `pom.xml` and `application.yaml` we provide configuration needed for mongodb database.
Start your database before running this example.

Example docker commands to start databases in temporary containers:

Mongodb:
```
docker run --rm --name mongo -p 27017:27017 mongo
```



## Building the Docker Image

```
docker build -t helidon-erp-api .
```

## Running the Docker Image

```
docker run --rm -p 8080:8080 helidon-erp-api:latest
```

Exercise the application as described above.
                                

## Run the application in Kubernetes

If you don’t have access to a Kubernetes cluster, you can [install one](https://helidon.io/docs/latest/about/kubernetes) on your desktop.

### Verify connectivity to cluster

```
kubectl cluster-info                        # Verify which cluster
kubectl get pods                            # Verify connectivity to cluster
```

### Deploy the application to Kubernetes

```
kubectl create -f app.yaml                              # Deploy application
kubectl get pods                                        # Wait for quickstart pod to be RUNNING
kubectl get service  helidon-erp-api                     # Get service info
kubectl port-forward service/helidon-erp-api 8081:8080   # Forward service port to 8081
```

You can now exercise the application as you did before but use the port number 8081.

After you’re done, cleanup.

```
kubectl delete -f app.yaml
```


## Building a Custom Runtime Image

Build the custom runtime image using the jlink image profile:

```
mvn package -Pjlink-image
```

This uses the helidon-maven-plugin to perform the custom image generation.
After the build completes it will report some statistics about the build including the reduction in image size.

The target/helidon-erp-api-jri directory is a self contained custom image of your application. It contains your application,
its runtime dependencies and the JDK modules it depends on. You can start your application using the provide start script:

```
./target/helidon-erp-api-jri/bin/start
```

### Ahead-of-Time (AOT) Cache
An ahead-of-time (AOT) cache is included in the custom image by default to improve application startup performance.
You can learn more about the AOT cache in the JDK documentation.

The AOT cache increases your image size to get these performance optimizations. It can be of significant size (tens of MB).
The size of the cache is reported at the end of the build output.

If you’d rather have a smaller image size (with a slightly increased startup time) you can skip the creation of the AOT
cache by executing your build like this:

```
mvn package -Pjlink-image -Djlink.image.aotCache=false
```

For more information on available configuration options see the helidon-maven-plugin documentation.
                                

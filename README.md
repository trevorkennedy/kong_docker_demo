# Kong Demo Using Docker
This is a demo of the [Kong API Gateway](https://konghq.com/) which exhibits how to run Kong in a Docker environment as a proxy for a public web service as well as a Node.js microservice.

## Requirements
  - Linux, macOS, Windows (tested on Windows 10)
  - [Docker](https://www.docker.com)

## Installation
In a terminal window, clone this repository:

```
https://github.com/trevorkennedy/kong_docker_demo
```

## Running

Change to your local repo directory:

```
cd kong_docker_demo
```

Start the demo using [Docker Compose](https://docs.docker.com/compose/):

```
docker-compose up -d  
```

Note that the ***-d*** flag will running the containers as a background process.  The compose file does 4 things: 

1.  Creates PostgreSQL container as a backend data store (configuration info) for Kong .
2.  Creates an ephemeral to initialize the Postgres database.
3.  Builds a container for the system time Node.js microservice.
4.  Runs the Kong API Gateway as a container.

## Verify

Verify the containers (*kong-api, kong-db, kong-app*) are all running by using:

```
docker ps
```

Verify that the Kong Gateway and Admin API are responding:

```
curl http://localhost:8000
curl http://localhost:8001
```

## Create a Service Proxy

Now, create a local service named ***ip-service*** that will forward requests to the [Big Cloud Data API](https://www.bigdatacloud.com/). This is a two part process as both the service  and then the route must be established like so:

```
curl -i -X POST --url http://localhost:8001/services/ --data 'name=ip-service' --data 'url=https://api.bigdatacloud.net/'
curl -i -X POST --url http://localhost:8001/services/ip-service/routes --data 'hosts[]=ip-service'
```

You can now test both the plubic API and the proxied service using cURL:
```
curl https://api.bigdatacloud.net/data/client-ip
curl http://localhost:8000/data/client-ip --header 'Host: ip-service'
```

## Create a Microservice Endpoint

Following the recipe above, we can build an endpoint for the Node.js microservice:

```
curl -i -X POST --url http://localhost:8001/services/ --data 'name=time-service' --data 'url=http://kong-app:3000'
curl -i -X POST --url http://localhost:8001/services/time-service/routes --data 'hosts[]=time-service'
```

Recall that this service is running in the *kong-app* container and is exposed to the isolated Docker network via port 3000 but callers cannot access it without going through the API gateway. We can reach the ***time-service*** using the *Host* header:

```
curl http://localhost:8000 --header 'Host: time-service'
```

Use this endpoint to see a list of all the services enabled thus far:

```
curl http://localhost:8001/services
```

## Enable Rate Limiting

Added rate limiting (a maximum of 5 requests per minute in this case) to a service is as simple as running a single command:

```
curl -X POST http://localhost:8001/services/time-service/plugins/ --data "name=rate-limiting" --data "config.minute=5" --data "config.policy=local"
```

If you hit the *time-service* endpoint again, you will see additional headers in the response indicating the rate limit and usage:

```
curl http://localhost:8000 --header 'Host: time-service'
```

## Add Authorization

Key-based authorization can be add to the *time-service* endpoint by enabling the ***key-auth*** plugin, creating a user (*QA* in this example) and then providing the access key:

```
curl -i -X POST --url http://localhost:8001/services/time-service/plugins/ --data 'name=key-auth'
curl -i -X POST --url http://localhost:8001/consumers/ --data "username=QA"
curl -i -X POST --url http://localhost:8001/consumers/QA/key-auth/ --data 'key=Hello_Kong!'
```

Test the *time-service* endpoint both with and without specifying an API key:

```
curl http://localhost:8000 --header 'Host: time-service'
curl http://localhost:8000 --header 'Host: time-service' --header "apikey: Hello_Kong!"
```

## Conclusion

You have now added key-based authorization and rate limiting to a microservice without writing a single line of code.

![The Kong Way](Kong%20Way.png)

## Clean Up

Tear everything down using:

```
docker-compose down
```

***

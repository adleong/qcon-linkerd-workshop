# Linkerd Workshop: Latency aware load-balancing

In this workshop we will see the benefits of latency aware load-balancing.  This
directory contains a `docker-compose.yml` file which defines:

* 10 simple backend servers
* Slow Cooker, a load generator which is configured to send requests to the servers
* Prometheus, to collect metrics from the servers and from Slow Cooker
* Grafana, to display the metrics in Prometheus

Slow Cooker uses a naive Round Robin algorithm to send an equal number of
requests to each backend server.

## Getting a Baseline

Start the above containers by running:

```bash
docker-compose build
docker-compose up -d
```

View the Grafana dashboard

```bash
open http://localhost:3000 # or docker ip address
```

Note down the following values:

* p50 latency: ____
* p95 latency: ____
* p99 latency: ____
* success rate: ____

Notice the distribution of request volume per instance.  Do some servers seem
to be serving more requests than others, or are they all roughly the same?

## Adding Latency Aware Load-Balancing with Linkerd

Add this Linkerd service to the `docker-compose.yml` file to add a Linkerd
container:

```yaml
  linkerd:
    image: buoyantio/linkerd:1.3.5
    ports:
      - 4140:4140
      - 9990:9990
    volumes:
      - ./linkerd.yml:/io/buoyant/linkerd/config.yml:ro
      - ./disco:/disco
    command:
      - "/io/buoyant/linkerd/config.yml"
```

Edit the Slow Cooker service in `docker-compose.yml` to send to 
`http://linkerd:4140` instead of `http://server:8501`.

Linkerd reads its configuration from `linkerd.yml`.  Edit `linkerd.yml` to use
`ewma` as the load-balancer instead of `p2c`.

Redeploy the containers and look at the Grafana dashboard again:

```bash
docker-compose down
docker-compose build
docker-compose up -d
open http://localhost:3000 # or docker ip address
```

Now note the following values:

* p50 latency: ____
* p95 latency: ____
* p99 latency: ____
* success rate: ____

Notice the distribution of request volume per instance.  Do some servers seem
to be serving more requests than others, or are they all roughly the same?

## Clean up

Stop and remove all running containers:

```bash
docker-compose down
```


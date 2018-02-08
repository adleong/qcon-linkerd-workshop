# Linkerd Workshop: Transparent TLS

In this workshop we will see how to use Linkerd to transparently encrypt
service-to-service communication with TLS.  This directory contains a
`docker-compose.yml` file which defines:

* A simple backend server
* Slow Cooker, a load generator which is configured to send requests to the backend server
* Tshark, a container which sniffs the network traffic between these services

It also contains some sample TLS credentials.

## Getting a Baseline

Start the above containers by running:

```bash
docker-compose build
docker-compose up -d
```

```
Slow_cooker --------> :8501 Server1
                         ^
                         |
                      tshark
```

Slow cooker is now sending plain, unencrypted traffic to the backend server
and tshark is spying on that traffic.  Use this command to see tshark's output:

```bash
docker-compose logs -f tshark | sed 's/.*HTTP\/1.1//'
```

This will print the content bodies of the HTTP responses from the backend
server.

## Adding Transparent TLS with Linkerd

Add two Linkerd services to the `docker-compose.yml` file, one that will run
alongside Slow Cooker to initiate TLS and one that will run alongside the
backend server that will terminate TLS.

```yaml
  linkerd-client:
    image: buoyantio/linkerd:1.3.5
    ports:
      - 4140:4140
      - 9990:9990
    volumes:
      - ./linkerd-client.yml:/io/buoyant/linkerd/config.yml:ro
      - ./disco:/disco
      - ./ca-chain.cert.pem:/ca-chain.cert.pem
    command:
      - "/io/buoyant/linkerd/config.yml"

  linkerd-server:
    image: buoyantio/linkerd:1.3.5
    ports:
      - 4141:4141
      - 9991:9991
    volumes:
      - ./linkerd-server.yml:/io/buoyant/linkerd/config.yml:ro
      - ./disco:/disco
      - ./cert.pem:/cert.pem
      - ./private.pk8:/private.pk8
    command:
      - "/io/buoyant/linkerd/config.yml"
```

Edit the Slow Cooker service in `docker-compose.yml` to send to 
`http://linkerd:4140` instead of `http://server1:8501`.

The instances read their configuration from `linkerd-client.yml` and
`linkerd-server.yml` respectively.  

Edit `linkerd-client.yml` to configure the client Linkerd to initiate TLS.  Add
`/ca-chain.cert.pem` as a `trustCert` and set the `commonName` to Linkerd.
These values tell Linkerd to use the CA in this directory to validate that
the server has a valid certificate for the name "linkerd".

Edit `linkerd-server.yml` to configure the server Linkerd to terminate TLS.  Set
`certPath` to `/cert.pem` and `keyPath` to `/private.pk8` to tell Linkerd to
use the certificate and private key from this directory.  These are valid
credentials for the common name "linkerd".

Finally, edit the tshark service in `docker-compose.yml` to read traffic on port
4141 instead of port 8501.

```
Slow_cooker ---> :4140 Linkerd-client ===> :4141 Linkerd-server ---> :8501 Server1
                                              ^
                                              |
                                           tshark

--- plaintext
=== TLS
```

Redeploy the containers and look at tshark's output again:

```bash
docker-compose down
docker-compose build
docker-compose up -d
docker-compose logs -f tshark | sed 's/.*HTTP\/1.1//'
```

Are the responses from the backend server still visisble?

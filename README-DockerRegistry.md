# Private Docker Registry

## Prepare the routable IP or Domain

Assume the IP we're going to use is: 104.197.113.190

## SSL Cert

Please change below settings accordingly if you want to use self-signed cert.
Or obtain a 3rd party signed cert directly.

```
mkdir ssl

REGISTRY_IP="104.197.113.190"

# Create the configuration file for openssl settings
cat <<EOF > ssl/insecure.cnf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = SG
ST = Singapore
L = Singapore
O = Pivotal
CN = *

[v3_req]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:TRUE
subjectAltName = @alt_names

[alt_names]
IP = $REGISTRY_IP
DNS.1 = 8.8.8.8

EOF

openssl req -x509 -batch -nodes -newkey rsa:2048 -keyout ssl/registry.key -out ssl/registry.cert -config ssl/insecure.cnf -days 3650

ls ssl/
insecure.cnf  registry.cert  registry.key
```


## Deploy Consource

Assume we have targeted `bosh` env.

```
$ bosh2 -e bosh -d docker-registry deploy docker-registry.yml \
    --vars-store _creds-docker-registry.yml \
    --var-file=REGISTRY_CERT=ssl/registry.cert \
    --var-file=REGISTRY_KEY=ssl/registry.key \
    -v REGISTRY_NW_NAME="network-z1-only" \
    -v REGISTRY_AZ_NAME=z1 \
    -v REGISTRY_INTERNAL_IP=10.0.100.20 \
    -v REGISTRY_EXTERNAL_IP=104.197.113.190
```

## Play With It

### Before Playing

As we're using self-signed cert for our private Docker Registry, some actions must be taken before further playing:

```
$ sudo vi /etc/default/docker
    DOCKER_OPTS="--insecure-registry 104.197.113.190:443"

$ REGISTRY_URL="104.197.113.190:443"
$ sudo mkdir -p /etc/docker/certs.d/${REGISTRY_URL}/
$ sudo cp ssl/registry.cert /etc/docker/certs.d/${REGISTRY_URL}/ca.crt
$ sudo service docker restart
```

> Note: If you're using a self-signed cert, remember to set the insecure registries before further actions. Otherwise you will encounter below errors:
```
$ docker push 104.197.113.190:443/busybox
The push refers to a repository [104.197.113.190:443/busybox]
Get https://104.197.113.190:443/v2/: x509: certificate signed by unknown authority
```

### Push Images to Docker Registry

As an example, let's push the `busybox` image to our newly built Docker Registry

```
$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
...
busybox                     latest              efe10ee6727f        2 months ago        1.13MB
...

$ docker tag busybox 104.197.113.190:443/busybox

$ docker push 104.197.113.190:443/busybox
The push refers to a repository [104.197.113.190:443/busybox]
08c2295a7fa5: Pushed
latest: digest: sha256:8573b4a813d7b90ef3876c6bec33db1272c02f0f90c406b25a5f9729169548ac size: 527

$ curl -k https://104.197.113.190:443/v2/_catalog
{"repositories":["busybox"]}
```

Enjoy!
# Minio

Minio is a S3 compatiple store

## Deploy

```
$ bosh -e sandbox -d minio deploy minio.yml \
    --vars-store _creds-minio.yml \
    -v MINIO_NW_NAME="network-z1-only" \
    -v MINIO_AZ_NAME=z1 \
    -v MINIO_STATIC_IP=10.0.100.14
```

## Access

URL for browser: 10.0.100.14

Instll `mc`, the Minio CLI and:
```
$ mc config host add minio http://10.0.100.14 admin <PASSWD> S3v4
$ mc mb minio/docker-images
$ mc ls minio/docker-images
```
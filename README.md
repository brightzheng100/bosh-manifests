
# BOSH Manifests & Howto Guide

## Preparation

1. Create BOSH Director

```
$ git clone https://github.com/cloudfoundry/bosh-deployment
$ bosh create-env bosh-deployment/bosh.yml \
    --state=state.json \
    --vars-store=creds.yml \
    -o bosh-deployment/gcp/cpi.yml \
    -v director_name=bosh-gcp \
    -v internal_cidr=10.0.100.0/22 \
    -v internal_gw=10.0.100.1 \
    -v internal_ip=10.0.100.6 \
    --var-file gcp_credentials_json=<CREDENTIAL JSON FILE> \
    -v project_id=<PROJECT ID> \
    -v zone=us-central1-a \
    -v tags=[internal,bosh] \
    -v network=bosh-sandbox \
    -v subnetwork=bosh-releases
```


2. Alias BOSH Env

```
$ bosh alias-env sandbox -e 10.0.100.6 --ca-cert <(bosh int ./creds.yml --path /director_ssl/ca)
    Using environment '10.0.100.6' as anonymous user

    Name      bosh-gcp
    UUID      07587dc1-30ea-4da6-a814-f46c0962d25c
    Version   262.3.0 (00000000)
    CPI       google_cpi
    Features  compiled_package_cache: disabled
            config_server: disabled
            dns: disabled
            snapshots: disabled
    User      (not logged in)

    Succeeded
```


3. Export & Login to the Director
```
$ export BOSH_CLIENT=admin && export BOSH_CLIENT_SECRET=`bosh int ./creds.yml --path /admin_password`
```


4. Prepare & Update Cloud Config

```
$ bosh -e sandbox update-cloud-config cloud-config-gcp.yml
```


5. Upload Stemcells

```
$ bosh -e sandbox upload-stemcell stemcells/light-bosh-stemcell-3421.11-google-kvm-ubuntu-trusty-go_agent.tgz
```


## Completed BOSH Manifests & Howto Guide

- [Vault](README-Vault.md#vault)
- [Concourse](README-Concourse.md#concourse)


# Ref

- https://github.com/cloudfoundry-community/vault-boshrelease
- https://github.com/making/bosh-manifests

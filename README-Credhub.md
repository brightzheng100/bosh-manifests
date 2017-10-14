
# Credhub


## Overview

There are several typical depolyment opstions:

1. Credhub, together with an embedded Database (e.g. Postgres)
2. CredHub, together with an embedded Database (e.g. Postgres) and UAA
3. Credhub only, assuming you have an existing database and UAA server deployed at known locations

This manifest deploys CredHub and a co-located PostgreSQL database, which is the option #1 above.

> Note: we're talking about an independent Credhub here, acting like an application friendly credential hub, instead of the one embedded in BOSH Director as the BOSH config server.


## Retrieve Two Certs Before Deploying Credhub

CredHub requires a UAA server to manage authentication.

1. JWT public signing key: uaa-jwt.public_key

It can be easily retrieved back from the var store file generated while spinning up BOSH.
`$ bosh int ../creds.yml --path=/uaa_jwt_signing_key/public_key`

2. UUA CA: uaa-tls.ca

It can be easily retrieved back from the var store file generated while spinning up BOSH.
`$ bosh int ../creds.yml --path=/uaa_service_provider_ssl/ca`


## Deploy Credhub

```
$ bosh -e sandbox -d credhub deploy credhub.yml \
    --vars-store _creds-credhub.yml \
    -v CREDHUB_AZ_NAME=z1 \
    -v CREDHUB_NW_NAME=network-z1-only \
    -v CREDHUB_INTERNAL_IP=10.0.100.12 \
    -v CREDHUB_EXTERNAL_IP=10.0.100.12 \
    -v CREDHUB_VM_TYPE=small \
    -v CREDHUB_DISK_TYPE=hd10g \
    --var-file=uaa-jwt.public_key=<(bosh int ../creds.yml --path=/uaa_jwt_signing_key/public_key) \
    --var-file=uaa-tls.ca=<(bosh int ../creds.yml --path=/uaa_service_provider_ssl/ca) \
    -v UUA-URL="https://10.0.100.6:8443"
```


## Post Actions

After deployment is done, enable it by following steps:
Ref: https://github.com/pivotal-cf/credhub-release/blob/master/docs/bosh-install-with-credhub.md#-updates-after-deployment

### Install CLIs

- UAA CLI
`$ gem install cf-uaac`

- Credhub CLI
```
$ wget https://github.com/cloudfoundry-incubator/credhub-cli/releases/download/1.4.1/credhub-linux-1.4.1.tgz \
    && tar xvf credhub-linux-1.4.1.tgz \
    && chmod +x credhub \
    && sudo mv credhub /usr/local/bin \
    && rm credhub-linux-1.4.1.tgz
```

### Create CredHub Clients/Users in UAA

```
$ uaac target https://10.0.100.6:8443 --skip-ssl-validation

$ export UAA_ADMIN_CLIENT_SECRET=`bosh int ../creds.yml --path=/uaa_admin_client_secret`
$ uaac token client get uaa_admin -s ${UAA_ADMIN_CLIENT_SECRET}

$ uaac context
    [0]*[https://10.0.100.6:8443]
    skip_ssl_validation: true

    [1]*[uaa_admin]
        client_id: uaa_admin
        access_token: XXX
        token_type: bearer
        expires_in: 43199
        scope: clients.read password.write clients.secret clients.write uaa.admin scim.write scim.read
        jti: ff01f1ce3fa04aee858e30b566153a18

$ uaac clients
  admin
    scope: uaa.none
    resource_ids: none
    authorized_grant_types: client_credentials
    autoapprove:
    authorities: bosh.admin
    lastmodified: 1507281939293
  bosh_cli
    scope: openid bosh.admin bosh.read bosh.*.admin bosh.*.read
    resource_ids: none
    authorized_grant_types: password refresh_token
    autoapprove:
    access_token_validity: 120
    refresh_token_validity: 86400
    authorities: uaa.none
    lastmodified: 1507281939486
  credhub_cli
    scope: credhub.read credhub.write
    resource_ids: none
    authorized_grant_types: password refresh_token
    autoapprove:
    access_token_validity: 60
    refresh_token_validity: 1800
    authorities: uaa.none
    lastmodified: 1507281938903
  director_to_credhub
    ...
  hm
    ...
  uaa_admin
    scope: uaa.none
    resource_ids: none
    authorized_grant_types: client_credentials
    autoapprove:
    authorities: clients.read password.write clients.secret clients.write uaa.admin scim.write scim.read
    lastmodified: 1507281938696

$ uaac client add concourse_client \
        --name "Concourse UUA Client" \
        --scope credhub.read,credhub.write \
        --authorized_grant_types client_credentials \
        --authorities oauth.login
    New client secret:  *********
    Verify new client secret:  *********
    scope: credhub.write credhub.read
    client_id: concourse_client
    resource_ids: none
    authorized_grant_types: client_credentials
    autoapprove:
    authorities: oauth.login
    name: Concourse UUA Client
    required_user_groups:
    lastmodified: 1507605793908
    id: concourse_client
```

> Note: the `admin` is not really UAA admin (as below), use `uaa_admin` instead.
```
$ export UAA_CLIENT_ADMIN=`bosh int ../creds.yml --path=/admin_password`
$ uaac token client get admin -s ${UAA_CLIENT_ADMIN}
    Successfully fetched token via client credentials grant.
    Target: https://10.0.100.6:8443
    Context: admin, from client admin
```

### Test Credhub Using CLI

```
$ credhub login -s https://10.0.100.12:8844 -u concourse_user --skip-tls-validation
    Warning: The targeted TLS certificate has not been verified for this connection.
    Warning: The --skip-tls-validation flag is deprecated. Please use --ca-cert instead.
    password: *********
    Setting the target url: https://10.0.100.12:8844
    Login Successful

$ credhub set --name="/testing/hello" --type="value" --value="hello world!"
    id: 72db1886-94ba-48a1-94a4-bd657bf62b74
    name: /testing/hello
    type: value
    value: hello world!
    version_created_at: 2017-10-09T07:02:32Z
$ credhub find --name-like 'hello'
    credentials:
    - name: /testing/hello
    version_created_at: 2017-10-09T07:02:32Z
```
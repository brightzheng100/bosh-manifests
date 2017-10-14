# Concourse

Here we provide two integration implementations:
- Concourse + Vault
- Concourse + Credhub

> Note: even the idea for both of the implementations is very similiar, I'd still separate the processes so that no confusion would be there.


## Concourse + Vault

### Configure Vault Server

#### Export VALUT_ADDR

`$ export VAULT_ADDR=http://10.0.100.10:8200 `

#### Create A Mount For Concourse

```
$ vault mount -path=/concourse -description="Secrets for concourse pipelines" generic
    Successfully mounted 'generic' at '/concourse'!
```

#### Create Policy File

```
$ vi misc/vault-policy-concourse.hcl
  path "concourse/*" {
    policy = "read"
    capabilities =  ["read", "list"]
  }
```

#### Register Policy

```
$ vault policy-write policy-concourse misc/vault-policy-concourse.hcl
    Policy 'policy-concourse' written.
$ vault policies
    default
    policy-concourse
    root
```

#### Choose An Authentication Method

- If Using appRole for Authentication (This is what I'm using)

1. Enable approle:
```
$ vault auth-enable approle
    Successfully enabled 'approle' at 'approle'!
```

2. Export the name of the role that you would like to use
`$ export ROLE_NAME=concourse-role`

3. Create a role and fetch the role-id:
```
$ vault write auth/approle/role/$ROLE_NAME secret_id_ttl=10m token_num_uses=10 token_ttl=20m token_max_ttl=30m period=768h secret_id_num_uses=20 policies=policy-concourse
    Success! Data written to: auth/approle/role/concourse-role

$ vault read -format=json auth/approle/role/$ROLE_NAME/role-id
    {
        "request_id": "2faeaaf3-552d-b59e-4b87-ea37323b64f4",
        "lease_id": "",
        "lease_duration": 0,
        "renewable": false,
        "data": {
            "role_id": "xxx"
        },
        "warnings": null
    }
```

4. Fetch the secret-id for the role created above:
```
$ vault write -format=json -f auth/approle/role/$ROLE_NAME/secret-id
    {
        "request_id": "e81dba2a-dbe2-71f3-612b-44fb724574bc",
        "lease_id": "",
        "lease_duration": 0,
        "renewable": false,
        "data": {
            "secret_id": "xxx",
            "secret_id_accessor": "xxx"
        },
        "warnings": null
    }
```

5. Verify whether the role_id/secret_id pair works:
```
$ export ROLE_ID=`vault read auth/approle/role/$ROLE_NAME/role-id | grep role_id | awk '{print $2}'`
$ export SECRET_ID=`vault write -f auth/approle/role/$ROLE_NAME/secret-id | grep secret_id | head -1 | awk '{print $2}'`
$ vault write auth/approle/login role_id=${ROLE_ID} secret_id=${SECRET_ID}
    Key            	Value
    ---            	-----
    token          	87768766-5a7d-d1b6-3766-e755c61bdc5b
    token_accessor 	1ede055a-391d-3bcc-a62c-152a71cc5ef7
    token_duration 	768h0m0s
    token_renewable	true
    token_policies 	[concourse default]
```

6. Copy the role-id and the secret-id values from above and set it in the concourse deployment manifest.
Refre to `ops-files/concourse-vault-approle.yml`.


- If Using Periodic Token for Authentication

1. Initialize Vault and create a periodic token using the new policy:

```
$ vault init
$ vault token-create --policy=policy-concourse -period="600h" -format=json
    {
        "request_id": "9d7e7d2e-eeb6-a6b0-b19e-3a57d5cbda16",
        "lease_id": "",
        "lease_duration": 0,
        "renewable": false,
        "data": null,
        "warnings": null,
        "auth": {
            "client_token": "xxx",
            "accessor": "xxx",
            "policies": [
                "default",
                "policy-concourse"
            ],
            "metadata": null,
            "lease_duration": 2160000,
            "renewable": true
        }
    }
```

2. Copy the token value from above and set it in the concourse deployment manifest.
Refre to `ops-files/concourse-vault-token.yml`.


### Deploy Concourse

```
$ bosh -e sandbox -d concourse deploy concourse.yml \
    -o ops-files/concourse-vault-approle.yml \
    --vars-store _creds-concourse_vault.yml \
    -v CONCOURSE_NW_NAME=network-z1-only \
    -v CONCOURSE_AZ_NAME=z1 \
    -v CONCOURSE_INTERNAL_IP=10.0.100.11 \
    -v CONCOURSE_EXTERNAL_IP=104.197.186.152 \
    -v CONCOURSE_VAULT_MOUNT=/concourse \
    -v VAULT_ADDR=http://10.0.100.10:8200
```

> Note: 
- There has one-VM version of manifast named `concourse-allinone.yml`
- The `_creds-concourse_vault.yml` has below credential values:

```
ui_password: 
postgres_password: 

# Must be a 16 or 32 byte AES key
encryption_key: 

# If using ops-files/concourse-vault-approle.yml
vault_role_id: 
vault_secret_id: 

# If using ops-files/concourse-vault-token.yml
#client_token: 
#accessor: 
```

### Deploy Concourse Pinelines

#### Populate Required Variables/Secrects

Credential Lookup Rules:
- /concourse/TEAM_NAME/PIPELINE_NAME/foo_param
- /concourse/TEAM_NAME/foo_param

Write secrets to Vault using the following syntax:

`$ vault write concourse/<team-name>/<pipeline-name/<variable-name> value=<variable-value>`
Or
`$ vault write concourse/<team-name>/<variable-name> value=<variable-value>`

We create below values:
```
$ vault write concourse/main/vault-testing/my-secret value="Hello World"
$ vault read concourse/main/vault-testing/my-secret
    Key             	Value
    ---             	-----
    refresh_interval	768h0m0s
    value           	Hello World
```

#### Deploy Sample Pipeline

1. Sample Pipeline

```
$ vi misc/vault-testing-pipeline.yml

jobs:
- name: hello-world
  plan:
  - task: say-hello
    params:
      MY_SECRET: ((my-secret))
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: ubuntu
      run:
        path: bash
        args:
        - -c
        - |
          echo "MY_SECRET is ${MY_SECRET}"
```

2. Set Pipeline

```
$ fly -t bosh set-pipeline -p vault-testing -c misc/vault-testing-pipeline.yml
```

3. Manually Trigger the "hello world" Job

You should get:
```
say-hello
Pulling ubuntu@sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641...
sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641: Pulling from library/ubuntu
...
Successfully pulled ubuntu@sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641.

MY_SECRET is Hello World
```

Done!



## Concourse + Credhub

### Create UAA Client for Credhub

```
$ uaac target https://10.0.100.6:8443 --skip-ssl-validation

$ export UAA_ADMIN_CLIENT_SECRET=`bosh int ../creds.yml --path=/uaa_admin_client_secret`
$ uaac token client get uaa_admin -s ${UAA_ADMIN_CLIENT_SECRET}

$ uaac client delete concourse_client
$ uaac client add concourse_client \
        --name "Concourse UUA Client" \
        --scope "" \
        --authorities credhub.read,credhub.write \
        --authorized_grant_types client_credentials,password,refresh_token \
        --access_token_validity 3600 \
        --refresh_token_validity 3600

    scope: uaa.none
    client_id: concourse_client
    resource_ids: none
    authorized_grant_types: refresh_token client_credentials password
    autoapprove:
    access_token_validity: 3600
    refresh_token_validity: 3600
    authorities: credhub.write credhub.read
    name: Concourse UUA Client
    required_user_groups:
    lastmodified: 1507622062733
    id: concourse_client
```

### Deploy Concourse

```
$ bosh -e sandbox -d concourse deploy concourse.yml \
    -o ops-files/concourse-credhub.yml \
    --vars-store _creds-concourse_credhub.yml \
    -v CONCOURSE_NW_NAME=network-z1-only \
    -v CONCOURSE_AZ_NAME=z1 \
    -v CONCOURSE_INTERNAL_IP=10.0.100.11 \
    -v CONCOURSE_EXTERNAL_IP=104.197.186.152 \
    -v CONCOURSE_CREDHUB_MOUNT="/concourse" \
    -v CREDHUB_URL="https://10.0.100.12:8844" \
    --var-file=CREDHUB_CA_CERT=<(bosh int ./_creds-credhub.yml --path=/credhub-tls/ca)
```

If you want to deploy in one single VM for testing purpose:

```
$ bosh -e sandbox -d concourse deploy concourse-allinone.yml \
    -o ops-files/concourse-allinone-credhub.yml \
    --vars-store _creds-concourse_credhub.yml \
    -v CONCOURSE_NW_NAME=network-z1-only \
    -v CONCOURSE_AZ_NAME=z1 \
    -v CONCOURSE_INTERNAL_IP=10.0.100.11 \
    -v CONCOURSE_EXTERNAL_IP=104.197.186.152 \
    -v CONCOURSE_CREDHUB_MOUNT="/concourse" \
    -v CREDHUB_URL="https://10.0.100.12:8844" \
    --var-file=CREDHUB_CA_CERT=<(bosh int ./_creds-credhub.yml --path=/credhub-tls/ca)
```

> Note: The `_creds-concourse_credhub.yml` has below credential values, please set them before running the command.

```
ui_password: 

# Must be a 16 or 32 byte AES key
encryption_key: 

credhub_uaa_client_id: 
credhub_uaa_client_secret: 
```

### Deploy Concourse Pinelines

#### Populate Required Variables/Secrects

Credential Lookup Rules:
- /concourse/TEAM_NAME/PIPELINE_NAME/foo_param
- /concourse/TEAM_NAME/foo_param

Write secrets to Credhub using the following syntax:

`$ credhub set --name="/concourse/<team-name>/<pipeline-name/<variable-name>" --type="<type>" --value="<variable-value>"`
Or
`$ credhub set --name="/concourse/<team-name>/<variable-name>" --type="<type>" --value="<variable-value>"`

We create below values:
```
$ credhub login -s https://10.0.100.12:8844 --client-name concourse_client --client-secret=<PASSWD> --skip-tls-validation
    Warning: The targeted TLS certificate has not been verified for this connection.
    Warning: The --skip-tls-validation flag is deprecated. Please use --ca-cert instead.
    password: *********
    Setting the target url: https://10.0.100.12:8844
    Login Successful

$ credhub delete --name="/concourse/main/credhub-testing/my-secret"
$ credhub set --name="/concourse/main/credhub-testing/my-secret" --type="value" --value="Hello World"
$ credhub find --name-like 'my-secret'
    credentials:
    - name: /concourse/main/credhub-testing/my-secret
    version_created_at: 2017-10-09T07:44:52Z
```

> Note: refer to [this](README-Credhub.md) for the context about Credhub URL and user `concourse_user`

#### Deploy Sample Pipeline

1. Sample Pipeline

```
$ vi misc/vault-testing-pipeline.yml

jobs:
- name: hello-world
  plan:
  - task: say-hello
    params:
      MY_SECRET: ((my-secret))
    config:
      platform: linux
      image_resource:
        type: docker-image
        source:
          repository: ubuntu
      run:
        path: bash
        args:
        - -c
        - |
          echo "MY_SECRET is ${MY_SECRET}"
```

2. Set Pipeline

```
$ fly -t bosh set-pipeline -p credhub-testing -c misc/vault-testing-pipeline.yml
```

3. Manually Trigger the "hello world" Job

You should get:
```
say-hello
Pulling ubuntu@sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641...
sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641: Pulling from library/ubuntu
...
Successfully pulled ubuntu@sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641.

MY_SECRET is Hello World
```

Done!



## Known Issues & Solutions

### Retrieving Vault Values but Got Code: 403. Errors: permission denied

**ERROR**
```
Finding variable 'my-secret': Error making API request.

URL: GET http://10.0.100.10:8200/v1/concourse/main/vault-test/my-secret
Code: 403. Errors:

* permission denied
```

**SOLUTION**

BOSH SSH into `web` node to check the `atc` logs, it's most likely caused by wrong role_id/secret_id or they're expired.


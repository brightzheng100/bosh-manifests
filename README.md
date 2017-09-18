
# BOSH Manifest & Howto Guide

- [Vault](#vault)
- [Concourse](#concourse)

## Preparation

1. Upload Stemcells

```
$ bosh -e sandbox upload-stemcell stemcells/light-bosh-stemcell-3421.11-google-kvm-ubuntu-trusty-go_agent.tgz
```

## Vault

Ref:
- https://github.com/cloudfoundry-community/vault-boshrelease

```
$ bosh -e sandbox -d vault deploy vault.yml \
    -v VAULT_NW_NAME=network-z1-only \
    -v VAULT_STATIC_IP=10.0.100.10 \
    -v VAULT_VM_TYPE=small \
    -v VAULT_AZ_NAME=z1
```

After deployment is done, enable it by following steps:
Ref: https://github.com/cloudfoundry-community/vault-boshrelease

```
$ wget https://releases.hashicorp.com/vault/0.8.2/vault_0.8.2_linux_amd64.zip
$ unzip vault_0.8.2_linux_amd64.zip
$ chmod +x vault && sudo mv vault /usr/local/bin/
$ vault -v
    Vault v0.8.2 ('9afe7330e06e486ee326621624f2077d88bc9511')

$ export VAULT_ADDR=http://10.0.100.10:8200
    Unseal Key 1: xxx
    Unseal Key 2: xxx
    Unseal Key 3: xxx
    Unseal Key 4: xxx
    Unseal Key 5: xxx
    Initial Root Token: xxx

    Vault initialized with 5 keys and a key threshold of 3. Please
    securely distribute the above keys. When the vault is re-sealed,
    restarted, or stopped, you must provide at least 3 of these keys
    to unseal it again.

    Vault does not store the master key. Without at least 3 keys,
    your vault will remain permanently sealed.
$ vault unseal
    Key (will be hidden):
    Sealed: true
    Key Shares: 5
    Key Threshold: 3
    Unseal Progress: 1
    Unseal Nonce: 1ffc83e5-2e5f-1038-4e68-0223f1544746
$ vault unseal
    Key (will be hidden):
    Sealed: true
    Key Shares: 5
    Key Threshold: 3
    Unseal Progress: 2
    Unseal Nonce: 1ffc83e5-2e5f-1038-4e68-0223f1544746
$ vault unseal
    Key (will be hidden):
    Sealed: false
    Key Shares: 5
    Key Threshold: 3
    Unseal Progress: 0
    Unseal Nonce:
$ vault auth
    Token (will be hidden):
    Successfully authenticated! You are now logged in.
    token: 5e3f7eba-2e27-fc74-7c55-f6084bad4b00
    token_duration: 0
    token_policies: [root]
```

Now, you can put secrets in the vault, and read them back out. For example:

```
$ vault write secret/test mykey=myvalue
    Success! Data written to: secret/test
$ vault read secret/test
    Key             	Value
    ---             	-----
    refresh_interval	768h0m0s
    mykey           	myvalue
$ vault delete secret/test
    Success! Deleted 'secret/test' if it existed.
```

Yeah! Vault is ready to rock.

> Note: this is just the basic setup, please refer to vault-boshrelease for more advanced topics like HA, zero-downtime upgrade etc.



## Concourse

Ref:
- https://github.com/rahul-kj/concourse-vault

### Configure Vault Server for Credential Management in Concourse

#### Export VALUT_ADDR

`$ export VAULT_ADDR=http://10.0.100.10:8200 `

#### Create a mount in value for use by Concourse pipelines

```
$ vault mount -path=/concourse -description="Secrets for concourse pipelines" generic
    Successfully mounted 'generic' at '/concourse'!
```

#### Create a policy file (e.g. policy.hcl) with the following content:

```
$ vi misc/vault-policy-concourse.hcl
  path "concourse/*" {
    policy = "read"
    capabilities =  ["read", "list"]
  }
```

#### Register the policy above with Vault:

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
    --vars-store _creds-concourse.yml \
    -v CONCOURSE_NW_NAME=network-z1-only \
    -v CONCOURSE_AZ_NAME=z1 \
    -v CONCOURSE_INTERNAL_IP=10.0.100.11 \
    -v CONCOURSE_EXTERNAL_IP=104.197.186.152 \
    -v CONCOURSE_VAULT_MOUNT=/concourse \
    -v VAULT_ADDR=http://10.0.100.10:8200
```

> Note: sample _creds-concourse.yml has below credential values:
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

#### Populate all the variables in Vault

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

#### Deploy Pipelines

Sample pipeline as below:

`$ vi misc/vault-testing-pipeline.yml`

```
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

```
$ fly -t bosh set-pipeline -p vault-testing -c misc/vault-testing-pipeline.yml
```

Manually trigger the "hello world" job and you should get:
```
say-hello
Pulling ubuntu@sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641...
sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641: Pulling from library/ubuntu
...
Successfully pulled ubuntu@sha256:2b9285d3e340ae9d4297f83fed6a9563493945935fc787e98cc32a69f5687641.

MY_SECRET is Hello World
```

Done!


# Known Issues & Solutions

## Retrieving Vault Values but Got Code: 403. Errors: permission denied

*ERROR*
```
Finding variable 'my-secret': Error making API request.

URL: GET http://10.0.100.10:8200/v1/concourse/main/vault-test/my-secret
Code: 403. Errors:

* permission denied
```

*SOLUTION*
BOSH SSH into `web` node to check the `atc` logs, it's proprably caused by wrong role_id/secret_id or they're expired.


# Ref

- https://github.com/making/bosh-manifests

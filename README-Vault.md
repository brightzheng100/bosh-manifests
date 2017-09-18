
# Vault

## Deploy Vault

```
$ bosh -e sandbox -d vault deploy vault.yml \
    -v VAULT_NW_NAME=network-z1-only \
    -v VAULT_STATIC_IP=10.0.100.10 \
    -v VAULT_VM_TYPE=small \
    -v VAULT_AZ_NAME=z1
```

## Post Actions

After deployment is done, enable it by following steps:
Ref: https://github.com/cloudfoundry-community/vault-boshrelease

1. Prepare Tools

```
$ wget https://releases.hashicorp.com/vault/0.8.2/vault_0.8.2_linux_amd64.zip
$ unzip vault_0.8.2_linux_amd64.zip
$ chmod +x vault && sudo mv vault /usr/local/bin/
$ vault -v
    Vault v0.8.2 ('9afe7330e06e486ee326621624f2077d88bc9511')
```

2. Unseal Vault

```
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

3. Try Putting Values

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

# HashiCorp Vault Helm Chart

This directory contains a Kubernetes chart to deploy HashiCorp Vault to GKE with KMS unsealing.

## Prerequisites Details

* Kubernetes 1.6+
* TLS certificates or a way to provision them 

## Chart Details

This chart will create the following Kubernetes resources:
* A service pointing to a TLS enabled listener on vault
* A statefulset of pods with a vault-init container that will unseal it from KMS

## GCP Configurations
1. Create a Data store bucket
```console
export PROJECT=myproject
export BUCKET=my-vault-bucket
$ gsutil mb -l eu -p ${PROJECT} gs://${BUCKET}
```

2. Set versioning on 
export
```console
$ gsutil versioning set on gs://${BUCKET}
```

3. Set lifecycle to 1 versions and delete the rest, create a file with this json

```json
{
  "rule": [
    {
      "action": {
        "type": "Delete"
      },
      "condition": {
        "numNewerVersions": 1
      }
    }
  ]
}
```
4. Set the lifecycle on the bucket

```console
$ gsutil lifecycle set lifecycle.json  gs://${BUCKET}
```
5. Enable the services for the project:

* cloudkms.googleapis.com
* cloudresourcemanager.googleapis.com
* container.googleapis.com
* compute.googleapis.com
* iam.googleapis.com
* logging.googleapis.com
* monitoring.googleapis.com

```console
$ gcloud services enable ${SERVICE}
```

6. Create the KMS keyring

export VAULT_NAME=vault
export KEY_NAME=vault-init
```console
$ gcloud kms keyrings create ${VAULT_NAME} --location global
$ gcloud kms keys create ${KEY_NAME} --location global --keyring ${VAULT_NAME} --purpose encryption
```
7. Check the keyring was created

```console
gcloud kms keyrings list --location global
```
8. Check the key was created

```console
gcloud kms keys list --keyring=${VAULT_NAME} --location=global
```
9. Create a service account and set this permissions

From https://github.com/sethvargo/vault-init :
```
To use this service, the service account must have the following minimum scope(s):

https://www.googleapis.com/auth/cloudkms
https://www.googleapis.com/auth/devstorage.read_write

Additionally, the service account must have the following minimum role(s):

roles/cloudkms.cryptoKeyEncrypterDecrypter
roles/storage.objectAdmin OR roles/storage.legacyBucketWriter
```

10. Give the service account permissions

```console
gcloud kms keys add-iam-policy-binding \
  --location global ${KEY_NAME}
  --keyring ${VAULT_NAME}
  --member serviceAccount:${SA_ACCOUNT}@${PROJECT}.iam.gserviceaccount.com \
  --role roles/cloudkms.cryptoKeyDecrypter
```

## Configuration

The following table lists the configurable parameters of the Vault chart and their default values.

|             Parameter             |              Description                 |               Default               |
|-----------------------------------|------------------------------------------|-------------------------------------|
| `replicaCount`                    | Number of replica pods to run            | `3`
| `vaultInit.repository`            | Repository part of the container image URL     | `sethvargo` |
| `vaultInit.name`                  | Name part of the container image URL           | `vault-init` |
| `vaultInit.tag`                   | Tag part of the container image URL      | `1.0.0` |
| `service.type`                    | Type of the Kubernetes service           | `ClusterIP` |
| `service.port`                    |                                          | `8200` |
| `vault.dev                        | Enable dev mode                          | `true` |
| `vault.extraEnv`                  | Configure additional environment variables for the Vault containers | `{}` |
| `vault.logLevel`                  | https://www.vaultproject.io/docs/commands/server.html#log-level | `info`|
| `vault.readiness.readyIfSealed`   | `false` |
| `vault.readiness.readyIfStandby`  | `true`| 
| `vault.readiness.readyIfUninitialized` | `true` |
| `vault.gcp.bucket` | `Data store bucket name` | `vault-bucket` |
| `vault.gcp.project` | `GCP project name` | `myproject` |
| `vault.gcp.service_account_key` | `JSON private key` | 
| `vault.gcp.kms.region` | `KMS region` | `global` |
| `vault.gcp.kms.key_ring` | |`vault` |
| `vault.gcp.kms.crypto_key`| | `vault-init` |
| `vault.gcp.kms.recovery_shares_num` | | `5` |
| `vault.gcp.kms.secret_recovery_threshold` | |`3` |
| `estafette.enabled` |`Enable estafette CD/CI support: https://estafette.io/usage/`  | `false` |
| `estafette.hostname` | `Hostname to use for LetEncrypt and Cloudflare` | `vault.example.com` |


## How to install it

```console
$ export KEY=$(base64 -w 0 MYKEY.json)
$ helm install vault-gcp-kms --name tooling --set vault.gcp.service_account_key=${KEY} 
$ helm upgrade tooling vault-gcp-kms --set vault.gcp.service_account_key=${KEY}
$ helm delete tooling --purge
```

This will name all resources as tooling-vault-gcp-kms

```console
$ helm list
NAME            REVISION        UPDATED                         STATUS          CHART                           APP VERSION     NAMESPACE    
tooling         1               Mon Mar 11 17:15:06 2019        DEPLOYED        vault-gcp-kms-0.1.0             1.0.3           monitoring
```
## Using Vault

Once the Vault pod is ready, it can be accessed using a `kubectl port-forward`:

```console
$ kubectl port-forward svc/vault-svc 8200
$ export VAULT_ADDR=https://127.0.0.1:8200
$ vault status --tls-skip-verify
```

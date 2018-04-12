# Vault Helm Chart

This directory contains a Kubernetes chart to deploy a Vault server.

## Prerequisites Details

* Kubernetes 1.6+

## Chart Details

This chart will do the following:

* Implement a Vault deployment

Please note that a backend service for Vault (for example, Consul) must
be deployed beforehand and configured with the `vault.config` option. YAML
provided under this option will be converted to JSON for the final Vault
`config.json` file.

> See https://www.vaultproject.io/docs/configuration/ for more information.

## Installing the Chart

To install the chart, use the following, this backs Vault with a Consul cluster:

```console
$ helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
$ helm install incubator/vault --set vault.dev=false --set vault.config.storage.consul.address="myconsul-svc-name:8500",vault.config.storage.consul.path="vault"
```

An alternative example using the Amazon S3 backend can be specified using:

```
vault:
  config:
    storage:
      s3:
        access_key: "AWS-ACCESS-KEY"
        secret_key: "AWS-SECRET-KEY"
        bucket: "AWS-BUCKET"
        region: "eu-central-1"
```

## Configuration

The following table lists the configurable parameters of the Vault chart and their default values.

|       Parameter         |           Description               |                         Default                     |
|-------------------------|-------------------------------------|-----------------------------------------------------|
| `image.pullPolicy`      | Container pull policy               | `IfNotPresent`                                      |
| `image.repository`      | Container image to use              | `vault`                                             |
| `image.tag`             | Container image tag to deploy       | `0.9.0`                                             |
| `vault.dev`             | Use Vault in dev mode               | true (set to false in production)                   |
| `vault.customSecrets`   | Custom secrets available to Vault   | `[]`                                                |
| `vault.config`          | Vault configuration                 | No default backend                                  |
| `replicaCount`          | k8s replicas                        | `1`                                                 |
| `resources.limits.cpu`  | Container requested CPU             | `nil`                                               |
| `resources.limits.memory` | Container requested memory        | `nil`                                               |
| `affinity`              | Affinity settings                   | See values.yaml                                               |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

## Using Vault

Once the Vault pod is ready, it can be accessed using a `kubectl
port-forward`:

```console
$ kubectl port-forward vault-pod 8200
$ export VAULT_ADDR=http://127.0.0.1:8200
$ vault status
```

### TLS config

This is example of running Vault with Kubernetes generated TLS certificate:

1. Create `CertificateSigningRequest` according to [documentation](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/):

1. Create `Secret` holding certificate and private key PEM:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: vault-tls
data:
  tls.crt: ${BASE64_ENCODED_CRT}
  tls.key: ${BASE64_ENCODED_PRIVATE_KEY}
```

1. Set up minimal `values.yml` configuration:

```yaml
vault:
  tls:
    enabled: true
  env:
  - name: VAULT_CAPATH
    value: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  customSecrets:
  - secretName: vault-tls
    mountPath: /vault/tls
  config:
    listener:
      tcp:
        tls_disable: false
        tls_cert_file: /vault/tls/tls.crt
        tls_key_file: /vault/tls/tls.key
        
# ingress:
#  enable: true
#  annotations:
#    kubernetes.io/ingress.class: "nginx"
#    nginx.ingress.kubernetes.io/secure-backends: "true"
```

### HA unseal script

Script for easier HA mode Vault unsealing:
```bash
#!/bin/sh
set -e
release=$1
namespace=$(helm status vault -o json | jq -r '.namespace')
read -rsp 'Vault Unseal Key: ' key
echo
for pod in $(kubectl -n ${namespace} get pods -l release=${release} -o json | jq -r '.items[] | select(.status.phase == "Running" and (.status.containerStatuses | any(.name == "vault" and (.ready | not)))) | .metadata.name')
do
    kubectl -n ${namespace} exec -ti ${pod} vault operator unseal "${key}"
done
```
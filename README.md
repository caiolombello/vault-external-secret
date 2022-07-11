# Vault and External Secrets with Kubernetes

## How-To

### Install Vault with Helm

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com

helm install vault hashicorp/vault
```

#### View the Vault UI

```shell
kubectl port-forward vault-0 8200:8200
```

#### Initialize and unseal Vault

##### CLI initialize and unseal

View all the Vault pods in the current namespace:

```shell
kubectl get pods -l app.kubernetes.io/name=vault
```

Initialize one Vault server with the default number of key shares and default key threshold:

```shell
kubectl exec -ti vault-0 -- vault operator init
```

The output displays the key shares and initial root key generated.

Unseal the Vault server with the key shares until the key threshold is met:

```shell
## Unseal the first vault server until it reaches the key threshold
kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 1
kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 2
kubectl exec -ti vault-0 -- vault operator unseal # ... Unseal Key 3
```

Repeat the unseal process for all Vault server pods. When all Vault server pods are unsealed they report READY `1/1`.

```shell
kubectl get pods -l app.kubernetes.io/name=vault

```

Login as root

```shell
kubectl exec -ti vault-0 -- vault login
```

### Install External Secret Operator with Helm

```shell
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

Validate

```shell
kubectl get pods -n "external-secrets"
```

Create a file `cluster-secret-store.yaml` with following content.

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault-server-internal.vault:8200"
      path: "secret"
      version: "v1"
      auth:
        tokenSecretRef:
          name: "vault-token"
          key: "token"
          namespace: vault
```

Create a file `vault-token-secret.yaml` with the following content, ensure `token: cy50cU4xVXFHUEpqdFpLeklMaW11VjdQZlg=` matches with your vault cluster root token.

To encode your vault cluster root token use:

```shell
echo -n $VAULT_TOKEN |base64
```

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
  namespace: vault
data:
  token: cy50cU4xVXFHUEpqdFpLeklMaW11VjdQZlg=
```

Create vault namespace

```shell
kubectl create namespace vault
```

Now apply both files as follows

```shell
kubectl apply -f cluster-secret-store.yaml
kubectl apply -f vault-token-secret.yaml
```

Create a file `external-pullsecret-cluster.yaml` with following content.

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: pullsecret-cluster-sno01
  namespace: sno01
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: pullsecret-cluster-sno01
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: secret/openshiftpullsecret
      property: dockerconfigjson
```

Enable the kv endpoint as follows

```shell
kubectl exec -it vault-0 -- vault login
kubectl exec -it vault-0 -- vault secrets enable -path=secret/ kv
```

Now add data

```shell
oc exec -it vault-0 -- vault kv put secret/pullsecret dockerconfigjson='{"auths":{"cloud.openshift.com":{"auth":"3BlbnNoaWZ0LXJl==","email":"example@redhat.com"},"quay.io":{"auth":"ZZMVhJRUJUR1I3WUwxN05VMQ==","email":"example@redhat.com"},"registry.connect.redhat.com":{"auth":"3BlbnNoaWZ0LXJl==","email":"example@redhat.com"},"registry.redhat.io":{"auth":"==","email":"example@redhat.com"}}}'
```

Now apply the `ExternalSecret` object created above.

```shell
kubectl apply -f external-pullsecret-cluster.yaml
```

#### Validating

- ExternalSecret

```shell
kubectl get externalsecret -n sno01 pullsecret-cluster-sno01
```

- Kubernetes secret object created by External Secret Operator.

```shell
kubectl get secrets -n sno01 pullsecret-cluster-sno01
```

As you can see, a new secret is present with the following content, where base64 data matches Vault secrets content.

```shell
kubectl get secrets -n sno01 pullsecret-cluster-sno01 -o yaml
```

You may compare the decoded secret data to the secret you created earlier; both should match.

### References:

[Vault Documentation](https://www.vaultproject.io/docs/platform/k8s/helm/run)
[Balkrishna Pandey](https://www.goglides.dev/bkpandey/how-to-manage-secrets-in-openshiftkubernetes-using-vault-and-external-secrets-1n6k)

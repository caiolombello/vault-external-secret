# Vault and External Secrets with Kubernetes

![HCP Vault](https://external-secrets.io/v0.5.1/pictures/diagrams-provider-vault.png)

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

### Enable Kubernetes Auth

```shell
kubectl exec -it vault-0 -- vault auth enable kubernetes
```

Configure the Authentication

```shell
# env
export KUBERNETES_PORT_443_TCP_ADDR=10.43.0.1 # found with `kubectl get svc`
export SERVICE_ACCOUNT_TOKEN=$(kubectl exec -it vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
export PATH_TO_CERTIFICATE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

```shell
kubectl exec -it vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host=$KUBERNETES_PORT_443_TCP_ADDR \
    token_reviewer_jwt=$SERVICE_ACCOUNT_TOKEN \
    kubernetes_ca_cert=@$PATH_TO_CERTIFICATE \
    disable_issuer_verification=true
```

Create a role

```shell
kubectl exec -it vault-0 -- vault write auth/kubernetes/role/internal-app \
  policies=internal-app \
  bound_service_account_namespaces=default \
  bound_service_account_names=k8s-service-acct \
  ttl=24h
```

Create a Vault Policy

```shell
kubectl exec -it vault-0 -- vault policy write internal-app - <<EOF
path "internal/data/database/config" {
  capabilities = ["read"]
}
EOF
```

Create Kubernetes Service Accounts

```shell
kubectl create sa k8s-service-acct \
  --namespace default
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
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://10.43.129.3:8200" # Pegue o IP do servidor através do comando `kubectl get svc`
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "internal-app"
          serviceAccountRef:
            name: "k8s-service-acct"
          secretRef:
            name: "kubernetes-jwt"
            key: "jwt"
            namespace: "kubernetes-jwt"
```

Encode JWT as Base64

```shell
echo -n $SERVICE_ACCOUNT_TOKEN |base64
```

Create a file `jwt-secret.yaml` with the following content

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-jwt
  namespace: kubernetes-jwt
data:
  jwt: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklsUmZVa1pVVUV4VUxYUnRhRkJOV2pobFREWlNWMVl6YkVORVNWY3RhbXd6V210R1FuYzNNVFZuY1VVaWZRLmV5SmhkV1FpT2xzaWFIUjBjSE02THk5cmRXSmxjbTVsZEdWekxtUmxabUYxYkhRdWMzWmpMbU5zZFhOMFpYSXViRzlqWVd3aUxDSnJNM01pWFN3aVpYaHdJam94TmpnNU1qVTRORE0yTENKcFlYUWlPakUyTlRjM01qSTBNellzSW1semN5STZJbWgwZEhCek9pOHZhM1ZpWlhKdVpYUmxjeTVrWldaaGRXeDBMbk4yWXk1amJIVnpkR1Z5TG14dlkyRnNJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5STZleUp1WVcxbGMzQmhZMlVpT2lKa1pXWmhkV3gwSWl3aWNHOWtJanA3SW01aGJXVWlPaUoyWVhWc2RDMHdJaXdpZFdsa0lqb2laV1k1TnpjeU9XSXRNREU1T1MwME5EbGlMV0psTnpJdFptSXpNekUxTURaallqQXlJbjBzSW5ObGNuWnBZMlZoWTJOdmRXNTBJanA3SW01aGJXVWlPaUoyWVhWc2RDSXNJblZwWkNJNkltUTVaREl6TWpNeUxUVmtZMll0TkdFeU15MDRORGcyTFdRMlpqRmtZV0V3TWpSaU9DSjlMQ0ozWVhKdVlXWjBaWElpT2pFMk5UYzNNall3TkROOUxDSnVZbVlpT2pFMk5UYzNNakkwTXpZc0luTjFZaUk2SW5ONWMzUmxiVHB6WlhKMmFXTmxZV05qYjNWdWREcGtaV1poZFd4ME9uWmhkV3gwSW4wLk02MnJEcXpJRzZPMFI1ZC1MQTAxcHVPdnNROVhtbzhhZWhVSmFuejl3MUdad0ljRmNkLXJpLVFKVEIzSnhRNWhHSGplWGtBd3VCVUIwTkZHWFZYS1l6cVF6SWRrTEpiZEF0WklYTUdGeXdvdThrOG1GazNfTm84UWdDZUgyeUpyUTUwVGZ4ZFhQbGxULTFLb1hjUklOU0VvR2d4QW51NUJpdzNYM09YZXpLSWVDTlRIQzRXc1Awa1ZoUjNBU2xPaDlUZi1BOU5DRjRTaGdGSWQ2NE1nWkxwX19iX0k0RHllTzM2dmVPcUVyY3BSOENTTTVvNXNGaU83elY1WDZiWEZnOFl0dWo0RGZONk4tUUZJZi1qQUxBY3o3UEpQb1hRWUdJNUM1cFNNMGlzMktVWlpVOGdxYU5sWlNObnhRVEFZSG5ZU3ZWWE0yMzVfV0NwQS1CU2Zidw==
```

Create jwt-secret namespace

```shell
kubectl create namespace kubernetes-jwt
```

Now apply both files as follows

```shell
kubectl apply -f jwt.yaml
kubectl apply -f cluster-secret-store.yaml
```

Create a file `external-pullsecret-cluster.yaml` with following content.

```yaml
apiVersion: external-secrets.io/v1beta1
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
      key: secret/kubernetespullsecret
      property: dockerconfigjson
```

Create namespace

```shell
kubectl create namespace sno01
```

Enable the kv endpoint as follows

```shell
kubectl exec -it vault-0 -- vault login
kubectl exec -it vault-0 -- vault secrets enable -path=secret/ kv
```

Now add data

```shell
kubectl exec -it vault-0 -- vault kv put secret/pullsecret dockerconfigjson='{"auths":{"cloud.openshift.com":{"auth":"3BlbnNoaWZ0LXJl==","email":"example@redhat.com"},"quay.io":{"auth":"ZZMVhJRUJUR1I3WUwxN05VMQ==","email":"example@redhat.com"},"registry.connect.redhat.com":{"auth":"3BlbnNoaWZ0LXJl==","email":"example@redhat.com"},"registry.redhat.io":{"auth":"==","email":"example@redhat.com"}}}'
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

## References:

[Vault Documentation](https://www.vaultproject.io/docs/platform/k8s/helm/run)

[Balkrishna Pandey](https://www.goglides.dev/bkpandey/how-to-manage-secrets-in-openshiftkubernetes-using-vault-and-external-secrets-1n6k)

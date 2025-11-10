# External Secrets Operator with Custom CA

Use External Secrets Operator against a secret store using a TLS certificate
signed by a custom certificate authority (CA).

This method requires that the CA is trusted by OpenShift per the
[Custom PKI Docs]. If you the Vault server is signed by the OpenShift cluster
ingress operator (default certificate on a fresh installed cluster), you will
need to add the ingress-operator TLS certificate to your cluster trust bundle.

**NOTE:** If you do follow the [Custom PKI Docs] steps, you must wait until the
cluster operators reconcile before deploying External Secrets Operator. Watch
the cluster operators status with: `watch -d oc get co`

This was specifically tested with Vault but should work for any secret provider
signed by a custom CA that OpenShift is configured to trust.

## Installing

1. (Optional) Add any custom CA's to the cluster proxy

```bash
-- vault CA
oc create configmap custom-ca --from-file=ca-bundle.crt=~/tmp/vault-certs/root/ca.crt -n openshift-config
oc patch proxy/cluster --type=merge --patch='{"spec":{"trustedCA":{"name":"custom-ca"}}}'
```

Let the Cluster Operators settle

```bash
oc adm wait-for-stable-cluster --minimum-stable-period=120s --timeout=15m
```

2. Create the external-secrets project:

```bash
oc new-project external-secrets
```

3. Create the ca-bundle config map:

```bash
oc create -f - << EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: ca-bundle
  namespace: external-secrets
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true'
EOF
```

4. Deploy External Secrets Operator from Helm.

- genericTargets RBAC in values.yaml
- ca-bundle.crt custom mount for Vault CA

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install \
    external-secrets \
    external-secrets/external-secrets \
    -f values.yaml \
    -n external-secrets \
    --set installCRDs=true
```

[Custom PKI Docs]: https://docs.openshift.com/container-platform/latest/networking/configuring-a-custom-pki.html


## Configure

e.g. ClusterSecretStore onto Hashi Vault using k8s auth

```bash
cat <<'EOF' | oc apply -f-
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
  namespace: vault
spec:
  provider:
    vault:
      server: "https://vault-internal.vault.svc:8200"
      path: "kv"
      version: "v2"
      auth:
        kubernetes:
          # Path where the Kubernetes authentication backend is mounted in Vault
          mountPath: "apps.sno.sandbox.opentlc.com-openshift-gitops"
          role: "vault"
          serviceAccountRef:
            name: vault
            namespace: openshift-gitops
            audiences:
            - vault # Required for Vault 1.21+
EOF
```

## Test examples

Test simple secret + custom resources.

- https://external-secrets.io/latest/provider/hashicorp-vault
- https://external-secrets.io/latest/guides/targeting-custom-resources

Secret

```bash
cat <<'EOF' | oc apply -f-
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  refreshInterval: "60s"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: example
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: kv/data/ocp/sno/openshift-gitops/llama-stack
EOF
```

ConfigMap

```bash
cat <<'EOF' | oc apply -f-
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: vault-example-cm
spec:
  refreshInterval: "60s"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: example
    manifest:
      apiVersion: v1
      kind: ConfigMap
  dataFrom:
  - extract:
      key: kv/data/ocp/sno/openshift-gitops/llama-stack
EOF
```

See the [test-eso-policy.yaml](test-eso-policy.yaml) for more complex example using ACM ConfigPolicy

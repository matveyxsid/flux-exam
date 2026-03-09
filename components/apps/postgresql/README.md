# How to install

## Multiple installations per Namespace

1. Create files at `custom-resources`:

    ```yaml
    clusters/<CLUSTER>/custom-resources/<NAMESPACE>/postgres/helm-release.yaml

    ---
    apiVersion: helm.toolkit.fluxcd.io/v2beta1
    kind: HelmRelease
    metadata:
      name: postgresql
      namespace: flux-system
    spec:
      interval: 5m
      targetNamespace: <NAMESPACE>
      storageNamespace: <NAMESPACE>
      chart:
        spec:
          version: <CHART_VERSION>
      values:
      ## Add your values here (merged with base)
    ```

    [Chart values](https://github.com/bitnami/charts/blob/main/bitnami/postgresql/values.yaml)

    We use only those variants for Auth

    - via k8s secret (for dev we can store it at bitbucket)

        ```yaml
        values:
          auth:
            enablePostgresUser: true
            existingSecret: postgresql-<PSQL_POSTFIX>
        ```

    - via vault

        ```yaml
        values:
          auth:
            enablePostgresUser: true
            postgresPassword: vault:kv/data/<PATH_TO_SECRET>#postgres
        ```

2. Just package it:

    ```yaml
    clusters/<CLUSTER>/custom-resources/<NAMESPACE>/postgres/kustomization.yaml

    ---
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
      - ../../../../../components/apps/postgresql
    patches:
      - path: helm-release.yaml
        target:
          kind: HelmRelease
    ```

3. Create files at `sync-code`:

```yaml
clusters/<CLUSTER>/sync-code/<NAMESPACE>/postgres/postgresql.yaml

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: postgres-<PSQL_POSTFIX>
  namespace: flux-system
spec:
  patches:
  - patch: |

      - op: replace
        path: /metadata/name
        value: postgres-<PSQL_POSTFIX>
      - op: replace
        path: /spec/releaseName
        value: postgres-<PSQL_POSTFIX>
    target:
      kind: HelmRelease
      name: postgresql
  interval: 5m
  dependsOn:
    - name: repos
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./clusters/<CLUSTER>/custom-resources/<NAMESPACE>/postgres
  prune: false
  wait: true
  timeout: 20m
  retryInterval: 5m
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-variables
        # Use this ConfigMap if it exists, but proceed if it doesn't.
        optional: false
    substitute:
      PSQL_NAME: <PSQL_POSTFIX>

```

## Additional info

This is BANNED! Because at Bitnami chart we have a few variants for change those values.
[global: section](https://github.com/bitnami/charts/blob/3006db4a348169f37bdb0264058b0e5ab3a17721/bitnami/postgresql/values.yaml#L30)

```yaml
## Bad example. We won't use it!

values:
  auth:
    enablePostgresUser: false
  global:
    postgresql:
      auth:
        postgresPassword: vault/...
```

We use that section:
[auth: section](https://github.com/bitnami/charts/blob/3006db4a348169f37bdb0264058b0e5ab3a17721/bitnami/postgresql/values.yaml#L124)

```yaml
## Good example

values:
  auth:
    enablePostgresUser: true
    existingSecret: ""
    postgresPassword: vault:kv/data/<PATH_TO_SECRET>#postgres
```

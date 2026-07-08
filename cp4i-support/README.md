# cp4i-support — supporting workloads for the CP4I demos, as GitOps

These are the manually-applied cluster manifests folded back into git so the whole demo is
reproducible. Argo CD reconciles each via the Applications in `argocd/`.

| Overlay | What | Namespace |
|---|---|---|
| `soapserver/` | The SOAP backend the ACE `create-customer` flow calls (`soapserver-nonsecure`). | tools |
| `ace-bar/` | The nginx BAR host that serves the `barURL` (RWX PVC, mounted read-write so the ACE pipeline can publish to it). | tools |
| `mq-spring-app/` | The MQ JMS app exposed through APIC on the "Expose MQ app APIs" page. | dev |
| `rbac/` | The ClusterRole letting Argo CD manage `appconnect.ibm.com` (IntegrationRuntimes) + the RoleBinding letting the `ci` pipeline SA `oc cp` into `tools`. | cluster / tools |

Apply the Applications with `oc apply -k cp4i-support/argocd`.

**Note on secrets:** the `soapserver` and `mq-spring-app` Deployments reference an image-pull
secret (`quay-pull`) that is created out of band; harden it as a SealedSecret for a from-scratch
cluster. Image tags are pinned to what was cluster-proven (`soapserver:v1`, `mq-spring-app:0.0.11`).

# Manually Install Vault with Integrated Storage

## Pre Req
* AKS Cluster

## Connect to your AKS Cluster
Go to the root of this repo ` cd ..` to get the AKS resource group and cluster name from our terraform outputs.  We will use this to connect to our AKS cluster.

```
MY_RG=$(terraform output resource_group_name)
MY_CN=$(terraform output azure_aks_cluster_name)

az login
az aks get-credentials --resource-group ${MY_RG} --name ${MY_CN}
kubectl get pods
```

## Install Vault with Raft backend (Manual Steps)
Lets jump back into this folder to keep things clean.  `cd vault-raft`
```
helm repo add hashicorp https://helm.releases.hashicorp.com

helm install vault hashicorp/vault \
  --set='server.image.repository=hashicorp/vault-enterprise' \
  --set='server.image.tag=1.4.2_ent' \
  --set='server.ha.enabled=true' \
  --set='server.ha.raft.enabled=true'

helm status vault
kubectl get pods
kubectl exec vault-0 -- vault status
```
### Initialize & Unseal Vault Master
```
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json

export VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
export VAULT_ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")

kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

### Join & Unseal Vault Cluster
```
kubectl exec -ti vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -ti vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec -ti vault-1 -- vault status


kubectl exec -ti vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
kubectl exec -ti vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec -ti vault-2 -- vault status


kubectl get pods
```

### Login and test Vault
Login to vault and save token to env var.
```
export VAULT_TOKEN=$(kubectl exec -ti vault-0 -- vault login ${VAULT_ROOT_TOKEN} -format="json" | jq -r ".auth.client_token")

kubectl exec -ti vault-0 -- vault operator raft list-peers
```

### Install license
Setup a port-forward tunnel from your machine (keep this process in forground)
```
kubectl port-forward vault-0 8200:8200
```
In a new terminal window source necessary environment info for vault.  Create the vault license key json payload.

vault-ent.hclic
```
{
  "text": "01ABCDEFG..."
}
```

Using the Vault API from your machine (port-forward tunnel) input YOUR_TOKEN and write the license
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  --request PUT \
  --data @vault-ent.hclic \
  http://127.0.0.1:8200/v1/sys/license
```

Read the license
```
curl \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  http://127.0.0.1:8200/v1/sys/license
```

### Vault - kubectl commands
Return pods Not Running
```
kubectl get pods -o json  | jq -r '.items[] | select(.status.phase != "Running") | .metadata.namespace + "/" + .metadata.name'
```

Return pods that are Running but sealed
```
kubectl get pods -o json  | jq -r '.items[] | select(.status.phase == "Running" and select(.metadata.labels."vault-sealed" == "true" )) | .metadata.namespace + "/" + .metadata.name'
```

Return pods that are Running but not initialized
```
kubectl get pods -o json  | jq -r '.items[] | select(.status.phase == "Running" and select(.metadata.labels."vault-initialized" == "false" )) | .metadata.namespace + "/" + .metadata.name'
```
Return Vault Pods that are Running and Not Ready 0/1.
```
kubectl get pods -o json  | jq -r '.items[] | select(.status.phase == "Running" and ([ .status.containerStatuses[] | select(.ready == false )] | length ) == 1 ) | .metadata.namespace + "/" + .metadata.name'
```

Return Vault Pods that are Running and Ready 1/1.
```
kubectl get pods -o json  | jq -r '.items[] | select(.status.phase == "Running" and ([ .status.containerStatuses[] | select(.ready == true )] | length ) == 1 ) | .metadata.namespace + "/" + .metadata.name'
```
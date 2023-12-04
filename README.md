# OceanProtocol Provider on Kubernetes

Deploy [OceanProtocol Provider](https://docs.oceanprotocol.com/developers/provider) on Kubernetes, for example on [minikube](https://minikube.sigs.k8s.io/docs/start/).

First of all, to enable nginx ingress, run:

```shell
minikube addons enable ingress
```

To run locally, configure in `/etc/hosts` the following entry pointing to the IP of the minikube cluster, which can be obtained by running `minikube ip`:

```
192.168.64.4    ipfs.local provider.local
```

Then, follow the steps below.

## Define dataspace namespace

```shell
kubectl create namespace dataspace
```

## Deploy IPFS to persist Compute-to-Data outputs and logs

```shell
kubectl apply -f ipfs/ipfs-data-pvc.yaml 
kubectl apply -f ipfs/ipfs-deployment.yaml
kubectl apply -f ipfs/ipfs-service.yaml
kubectl apply -f ipfs/ipfs-ingress.yaml
```

Replace `ipfs.local` in [ipfs-ingress.yaml](ipfs/ipfs-ingress.yaml) with the hostname you are deploying the service to. Also replace the storage class in [ipfs-data-pvc.yaml](ipfs/ipfs-data-pvc.yaml) with the one you are using in your cluster.

## Deploy Postgres to manage computation jobs requests

```shell
kubectl apply -f postgres/postgres-configmap.yaml
kubectl apply -f postgres/postgres-data-pvc.yaml 
kubectl apply -f postgres/postgres-deployment.yaml
kubectl apply -f postgres/postgres-service.yaml
```

Replace the storage class in [postgres-data-pvc.yaml](postgres/postgres-data-pvc.yaml) with the one you are using in your cluster.

## Deploy the OceanProtocol Provider

```shell
kubectl apply -f provider/provider-deployment.yaml
kubectl apply -f provider/provider-service.yaml
kubectl apply -f provider/provider-ingress.yaml
```

Replace `provider.local` in [provider-ingress.yaml](provider/provider-ingress.yaml) with the hostname you are deploying the service to. 

Moreover, don't forget to replace the `PROVIDER_PRIVATE_KEY` and `UNIVERSAL_PRIVATE_KEY` ENV vars in [provider-deployment.yaml](provider/provider-deployment.yaml) with private keys only known to you.

## Deploy the OceanProtocol Operator Engine

```shell
kubectl apply -f operator-engine/operator-engine-service-account.yaml
kubectl apply -f operator-engine/operator-engine-role.yaml
kubectl apply -f operator-engine/operator-engine-role-binding.yaml
kubectl apply -f operator-engine/operator-engine-deployment.yaml
```

Update `IPFS_OUTPUT_PREFIX` and `IPFS_LOG_PREFIX` ENV vars in [operator-engine-deployment.yaml](operator-engine/operator-engine-deployment.yaml) with the hostname you deployed the IPFS service to. And use a `STORAGE_CLASS` that is available in your cluster and, preferable, that is automatically released and deleted after its use.

Moreover, don't forget to replace the `OPERATOR_PRIVATE_KEY` ENV var in [operator-engine-deployment.yaml](operator-engine/operator-engine-deployment.yaml) with a private key only known to you.

## Deploy the OceanProtocol Operator API

```shell
kubectl apply -f operator-api/operator-api-deployment.yaml
kubectl apply -f operator-api/operator-api-service.yaml
```

Update the `ALLOWED_PROVIDERS` and `OPERATOR_ADDRESS` ENV vars in [operator-api-deployment.yaml](operator-api/operator-api-deployment.yaml) with the public keys of the Provider and Operator Engine you previously deployed.

## Initialize Postgres

Finally, initialize the Postgres database through the Operator API from the Provider pod. Note that the Operator API is not visible externally. To do so, first open a shell in the Provider pod:

```shell
kubectl exec $(kubectl get pods -l=app=provider -o name) -n dataspace -it -- bash
```

From the shell in the Provider pod, first, install the curl command:

```shell
apt-get update && apt-get install curl -y
```

Then, call `pgsqlinit` in the Operator API using the `Admin` header to authenticate with one of the ALLOWED_ADMINS defined in the corresponding Operator API ENV var in [operator-api-deployment.yaml](operator-api/operator-api-deployment.yaml)):

```shell
curl -X POST "http://operator-api.dataspace:8050/api/v1/operator/pgsqlinit" -H  "accept: application/json" -H "Admin: postgresadmin"
```

## Test Provider Availability

To check that the Provider is available, visit the Provider URL in your browser, for instance:
http://provider.local

You should see the Provider API documentation:

```json
{
  "chainIds": [
    100
  ],
  "providerAddresses": {
    "100": "0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf"
  },
  "serviceEndpoints": {
    "computeDelete": [
      "DELETE",
      "/api/services/compute"
    ],
    "computeEnvironments": [
      "GET",
      "/api/services/computeEnvironments"
    ],
    "computeResult": [
      "GET",
      "/api/services/computeResult"
    ],
    "computeStart": [
      "POST",
      "/api/services/compute"
    ],
    "computeStatus": [
      "GET",
      "/api/services/compute"
    ],
    "computeStop": [
      "PUT",
      "/api/services/compute"
    ],
    "create_auth_token": [
      "GET",
      "/api/services/createAuthToken"
    ],
    "decrypt": [
      "POST",
      "/api/services/decrypt"
    ],
    "delete_auth_token": [
      "DELETE",
      "/api/services/deleteAuthToken"
    ],
    "download": [
      "GET",
      "/api/services/download"
    ],
    "encrypt": [
      "POST",
      "/api/services/encrypt"
    ],
    "fileinfo": [
      "POST",
      "/api/services/fileinfo"
    ],
    "initialize": [
      "GET",
      "/api/services/initialize"
    ],
    "initializeCompute": [
      "POST",
      "/api/services/initializeCompute"
    ],
    "nonce": [
      "GET",
      "/api/services/nonce"
    ],
    "validateContainer": [
      "POST",
      "/api/services/validateContainer"
    ]
  },
  "software": "Provider",
  "version": "2.1.6"
}
```

Which can be used to list the available compute environments:
http://provider.local/api/services/computeEnvironments

Providing something like:

```json
{
  "100": [
    {
      "consumerAddress": "0x7E5F4552091A69125d5DfCb7b8C2659029395Bdf",
      "cpuNumber": "2",
      "cpuType": "",
      "currentJobs": 0,
      "desc": "",
      "diskGB": "1",
      "feeToken": "0x0995527d3473b3a98c471f1ed8787acd77fbf009",
      "gpuNumber": "0",
      "gpuType": "",
      "id": "dataspace",
      "lastSeen": "1701725217.16205",
      "maxJobDuration": "900",
      "maxJobs": "1",
      "priceMin": "0.01",
      "ramGB": "4",
      "storageExpiry": 0
    }
  ]
}
```

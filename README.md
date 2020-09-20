# BESU + ALETHIO LITE EXPLORER + HELM + HELMFILE

This Helmfile repo deploys an Hyperledger Besu node connected to the Gorli testnet and an Alethio Lite Explorer connected to that Node.


## Requirements

* Helm (v3)
* Helmfile
* a Kubernetes cluster **(currently only tested on GKE)**

## Deployment

```bash
 $ helmfile sync
```

Ideally, that should be it.

**However**, the explorer chart relies on Helm's [lookup function](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/#using-the-lookup-function) to determine the node Service's external LoadBalancer IP. In some cases the LoadBalancer IP is not yet available while rendering the explorer chart and a second `helmfile sync` is necessary.


This deploys:

1. a Service (type=LoadBalancer) that exposes the RPC port (default=8545) on the node service loadbalancer's external IP. 
```bash
# to get the IP:PORT tuple:
  export NAMESPACE=<my_namespace> # by default this repo uses 'assignment' as service name
  export SERVICE=<my_node_rpc_servicename> # by default this repo uses 'besu-goerli-node' as service name
  kubectl get svc -n $NAMESPACE $SERVICE -o jsonpath='{.status.loadBalancer.ingress[0].ip}:{.spec.ports[?(@.name=="rpc")].port}'
```

2. an ingress-nginx controller to handle ingress traffic.

3. an Alethio Lite Explorer that is accessible through an Ingress on a subpath '/explorer':
```bash
# to get the IP tuple:
  export NAMESPACE=<my_namespace>
  export INGRESS=<my_ingressname> # by default this repo uses 'explorer-alethio-lite-explorer' as ingress name
  export EXPLORER_IP=$(kubectl get ing -n $NAMESPACE $INGRESS -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

The explorer is available on `${EXPLORER_IP}/explorer`.

## Caveats

There are some scenario's that can cause the explorer to lack data.

### RPC connection refused

It may take a while for the GCP LoadBalancer to (reliably) accept traffic on the Besu's RPC port.

**Diagnostics**

Verify locally if the Besu node is up and connected to Goerli:

```bash
 export NAMESPACE=<my_namespace> # by default this repo uses 'assignment' as service name
 export SERVICE=<my_node_rpc_servicename> # by default this repo uses 'besu-goerli-node' as service name

 export EXTENRAL_ADDR=$(kubectl get svc -n $NAMESPACE $SERVICE -o jsonpath='{.status.loadBalancer.ingress[0].ip}:{.spec.ports[?(@.name=="rpc")].port}')


 curl -X POST --data {"jsonrpc": "2.0", "method": "eth_chainId", "params":[], "id":1} ${EXTERNAL_ADDR}
{
  "jsonrpc" : "2.0",
  "id" : 1,
  "result" : "0x5"
}

```

If that fails, verify if the service responds locally:
```bash
 export NAMESPACE=<my_namespace> # by default this repo uses 'assignment' as service name
 kubectl port-forward -n $NAMESPACE  $(kubectl get pods -n $NAMESPACE -lapp.kubernetes.io/instance=node -o jsonpath="{.items[0].metadata.name}") 8080:8545 
 # in another terminal:
 curl -X POST --data {"jsonrpc": "2.0", "method": "eth_chainId", "params":[], "id":1} localhost:8080
 {
   "jsonrpc" : "2.0",
   "id" : 1,
   "result" : "0x5"
 }
```


**Workaround**

None really, seems to be a GCP glitch.

### Sync not active

It may take a long time for the node to be connected to sufficient peers and start syncing.

**Diagnostics**

Besu node log showing `| main | DEBUG | FastSyncActions | Waiting for at least 5 peers.`.

Decreasing the fast-sync-min-peers parameter and restarting the Besu node pod can initiate the sync.

**Workaround**

Pass in extra config via `config.extra` dict:
```bash
 $ helmfile sync --set config.extra.fast-sync-min-peers=1
```

Restart the Besu node pod:
```bash
 $ kubectl delete pod -n $NAMESPACE -lapp.kubernetes.io/instance=node
```

### Besu node's Kubernetes NAT Manager failure

The Kubernetes NAT manager attempts to determine its external IP using the Service's metadata.
In some cases the LoadBalancer IP is not yet available while starting the pod hence the NAT manager fails to setup its NAT configuration.

**Diagnostics**

Besu node log showing `| main | DEBUG | NatService | Nat manager failed to configure itself due to the following reason: Failed update information using pod metadata: null. NONE mode will be used`.

**Workaround**

Restart the Besu node pod:
```bash
 $ kubectl delete pod -n $NAMESPACE -lapp.kubernetes.io/instance=node
```



## Running conteiner docker
Create and run a particular image, possibly replicated.

Creates a deployment or job to manage the created container(s).
```bash
run NAME --image=image [--env="key=value"] [--port=port] [--replicas=replicas] [--dry-run=bool] [--overrides=inline-json] [--command] -- [COMMAND] [args...]
```

## Port Forward
Forward one or more local ports to a pod. This command requires the node to have 'socat' installed.

Use resource type/name such as deployment/mydeployment to select a pod. Resource type defaults to 'pod' if omitted.

If there are multiple pods matching the criteria, a pod will be selected automatically. The forwarding session ends when the selected pod terminates, and rerun of the command is needed to resume forwarding.


```bash
port-forward TYPE/NAME [options] [LOCAL_PORT:]REMOTE_PORT [...[LOCAL_PORT_N:]REMOTE_PORT_N]
```

## Querying Deployments

### Info
```bash
kubectl get deployments
```
### Verbose info
```bash
kubectl describe deployments/<name>
```


### Querying Cluster

```bash
kubectl get nodes
```


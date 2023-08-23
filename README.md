# Deploying Spinnaker in a Kubernetes cluster.
### Author: _Ved Prakash._ 

### Official Documents

| Document Type | Url |
| ------ | --------- |
| Spinnaker Installation | https://spinnaker.io/docs/setup/install/  |
| Spinnaker Architecture | https://spinnaker.io/docs/reference/architecture/microservices-overview/ |
| Minio Chart | https://github.com/minio/minio/tree/master/helm/minio |
| FE Plugin Guide | https://spinnaker.io/docs/guides/developer/plugin-creator/plugin-frontend/ |
| Nomad Docs | https://www.nomadproject.io/ |
| Nomad Plugin | https://github.com/spinnaker-plugin-examples/nomadPlugin |

> Note - Below Command is tested in Mac OS

### Pre-requisites
- At least 4 cores and 16GB of RAM available in the cluster.
- Docker Desktop
- Install Halyard in your system
- Install helm in your system

#### Start Halyard Container

```
mkdir ~/.hal
```
```
docker run --name halyard -v [.hal path]:/home/spinnaker/.hal -v [.kube config path]:/home/spinnaker/.kube/config -d gcr.io/spinnaker-marketplace/halyard:stable
```

> Note: `[.hal path]` = `/Users/[UserName]/.hal`
> `[.kube config path]` = `/Users/[UserName]/.kube/config`


#### Get a shell into Halyard container

```
docker exec -it halyard bash
```
#### Check if you can run kubectl commands

```
kubectl cluster-info
```

#### Set Kubernetes as the cloud provider

```
hal config provider kubernetes enable
```

#### Add a kubernetes account

```
hal config provider kubernetes account add my-k8s --provider-version v2 --context $(kubectl config current-context)
```

> Note: `my-k8s` = You can use other name also 

#### Enable artifacts

```
hal config features edit --artifacts true
```

#### Choose where to install Spinnaker

```
hal config deploy edit --type distributed --account-name my-k8s
```

### Below command needs to be run on your local machine where you have helm binary
#### Install minio in kubernetes cluster

- First Create the name space eg: `spinnaker`
  
```
kubectl create ns spinnaker
```

- Below command is to install the [minio] in local
  
```
helm repo add minio https://helm.min.io/
```

```
helm install minio --namespace spinnaker --set accessKey="myaccesskey" --set secretKey="mysecretkey" --set persistence.enabled=false minio/minio
```

### Back inside the halyard container
#### For minio, disable s3 versioning

```
mkdir ~/.hal/default/profiles
```

```
echo "spinnaker.s3.versioning: false" > ~/.hal/default/profiles/front50-local.yml
```

### Run below command to find the ip address 

```
kubectl get pods -n kube-system -o wide
```

### O/P of the above command will be something like below

```
NAME                                     READY   STATUS    RESTARTS         AGE   IP             NODE             NOMINATED NODE   READINESS GATES
kube-controller-manager-docker-desktop   1/1     Running   38 (6h47m ago)   26d   19x.xxx.xx.x   docker-desktop   <none>           <none>
```

### Edit the `.kube/config` file like below

```
vi .kube/config
```

> Change -  `server: https://127.0.0.1:6443` to `server: https://19x.xxx.xx.x:6443`

### Run the Below command to see the PODS in spinnaker NS

```
kubectl get pods -n spinnaker -o wide
```

### O/P of the above command will be something like below

```
NAME                                READY   STATUS             RESTARTS         AGE    IP          NODE             NOMINATED NODE   READINESS GATES
minio-67f7d447b7-95p2v              1/1     Running            3 (6h52m ago)    5d9h   10.x.x.xx   docker-desktop   <none>           <none>
```

> Note -  Above IP address will be the endpoint IP for minio `10.x.x.xx`

#### Set the storage type to minio/s3

```
hal config storage s3 edit --endpoint http://10.x.x.xx:9000 --access-key-id "myaccesskey" --secret-access-key "mysecretkey"
```

```
hal config storage s3 edit --path-style-access true
```

```
hal config storage edit --type s3
```

#### Choose spinnaker version to install

```
hal version list
```

```
hal config version edit --version <desired-version>
```

#### All Done! Deploy Spinnaker in Kubernetes Cluster

```
hal deploy apply
```

#### Wait for some time and Run the below commannd to see wether all service is running

```
kubectl get pods -n spinnaker -o wide
```

### O/P of the above command will be something like below

```
NAME                                READY   STATUS             RESTARTS         AGE    IP          NODE             NOMINATED NODE   READINESS GATES
minio-67f7d447b7-95p2v              1/1     Running            3 (6h52m ago)    2h   10.x.x.60   docker-desktop   <none>           <none>
spin-clouddriver-5cc6669746-rd6kd   1/1     Running            1 (6h52m ago)    2h     10.x.x.66   docker-desktop   <none>           <none>
spin-deck-5f9fbb879c-r4xqh          1/1     Running            3 (6h52m ago)    2h   10.x.x.54   docker-desktop   <none>           <none>
spin-echo-65c7bf46d9-7pkwt          0/1     Running            29 (8m12s ago)   2h   10.x.x.57   docker-desktop   <none>           <none>
spin-front50-9dcfd8c46-dz9km        0/1     Running            48 (4m15s ago)   2h     10.x.x.63   docker-desktop   <none>           <none>
spin-gate-5ff4574d65-dx8dd          1/1     Running            3 (6h52m ago)    2h   10.x.x.62   docker-desktop   <none>           <none>
spin-orca-679949d9-svrvv            1/1     Running            3 (6h52m ago)    2h   10.x.x.58   docker-desktop   <none>           <none>
spin-redis-6dd5fd554f-hssvk         1/1     Running            3 (6h52m ago)    2h   10.x.x.59   docker-desktop   <none>           <none>
spin-rosco-6cdf7f8d57-4kzs7         1/1     Running            3 (6h52m ago)    2h   10.x.x.65   docker-desktop   <none>           <none>
```

> Note -  All Service of the Spinnaker should be in 1/1 state- If any of the Service is failing then Try to see that service Logs in docker

#### Change the service type to either Load Balancer or NodePort

```
kubectl -n spinnaker edit svc spin-deck
```

#### Scroll down and try to update below section

```sh
    ports:                        
  - port: 9000                  
    protocol: TCP               
    targetPort: 9000            
    nodePort: 32323             
  selector:                  
    app: spin                
    cluster: spin-deck       
  sessionAffinity: None
  type: NodePort     
```

> Node- Change `type` from `ClusterIp` to `NodePort` and add `nodePort` as `32323`

```
kubectl -n spinnaker edit svc spin-gate
```

#### Scroll down and try to update below section

```sh
    ports:                        
  - port: 9000                  
    protocol: TCP               
    targetPort: 9000            
    nodePort: 32324            
  selector:                  
    app: spin                
    cluster: spin-deck       
  sessionAffinity: None
  type: NodePort     
```

> Node- Change `type` from `ClusterIp` to `NodePort` and add `nodePort` as `32324`

### Run below command to find the ip address 

```
kubectl get pods -n kube-system -o wide
```

### O/P of the above command will be something like below

```
NAME                                     READY   STATUS    RESTARTS         AGE   IP             NODE             NOMINATED NODE   READINESS GATES
kube-controller-manager-docker-desktop   1/1     Running   38 (6h47m ago)   26d   19x.xxx.xx.x   docker-desktop   <none>           <none>
```

#### Update config and redeploy'

```
hal config security ui edit --override-base-url "http://<worker-node-ip>:<nodePort>"
```

> Node-  `worker-node-ip` will be  `19x.xxx.xx.x` from  `kubectl get pods -n kube-system -o wide
` command `nodePort` for `ui` will be `32323`

```
hal config security api edit --override-base-url "http://worker-node-ip>:<nodePort>"
```

> Node-  `worker-node-ip` will be  `19x.xxx.xx.x` from  `kubectl get pods -n kube-system -o wide
` command `nodePort` for `api` will be `32324`

```
hal deploy apply
```

#### Update config and redeploy

```
hal config security ui edit --override-base-url "http://<LoadBalancerIP>:9000"
```

```
hal config security api edit --override-base-url "http://<LoadBalancerIP>:8084"
```

```
hal deploy apply
```

### After Deployment run below link in Browser

```
http://localhost:32323/
```

### If above link does not load the spinnaker UI then do changes in `.hal/config`

```
vi .hal/config or vi [path to hal config file]
```

#### Update the Below things

From
```
  security:
    apiSecurity:
      ssl:
        enabled: false
      overrideBaseUrl: http://worker-node-ip:32324
    uiSecurity:
      ssl:
        enabled: false
      overrideBaseUrl: http://worker-node-ip:32323
    authn:
```
To
```
  security:
    apiSecurity:
      ssl:
        enabled: false
      overrideBaseUrl: http://localhost:32324
    uiSecurity:
      ssl:
        enabled: false
      overrideBaseUrl: http://localhost:32323
    authn:
```

### Again Redeploy and Load the UI and check wether all pages is loading 

### If facing `CORS Error` then run below command

```
hal config security ui edit --corsAccessPattern 'http://localhost'
```
### Again Redeploy and Load the UI and check wether all pages is loading 

## License

MIT


[minio]: <https://github.com/minio/charts>




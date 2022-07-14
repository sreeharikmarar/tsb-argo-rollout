# TSB + ArgoRollout for Canary deployments

TSB mesh configurations integrated with argoCD and argoRollout for Canary deployment workflow 

### Install ArgoCD

Please refer their offical documentation [here](https://argo-cd.readthedocs.io/en/stable/getting_started/)

### Install ArgoRollout

Please refer their offical documentation [here](https://argoproj.github.io/argo-rollouts/installation/)

### Deploy Application 

You can choose to keep your existing k8s application deployment and service along with argo rollout object. You can make necassary changes to rollout object and Istio VirtualService/DesintationRule TSB configuration to achieve canary rollout. 

Refer application deployment spec [here](https://github.com/sreeharikmarar/tsb-argo-rollout/tree/main/application)

![Screenshot 2022-07-13 at 11 00 33 PM](https://user-images.githubusercontent.com/855824/178796152-d78b8201-82e0-4881-8c94-648f4afb2198.png)

### Deploy TSB mesh configurations

With the latest gitOps support in TSB 1.5, You can apply any TSB objects in ControlPlane cluster along with your application deployment manifest. Since ArgoRollout require you to make some modifications on VS/DR according to their Canary deployment strategy convention for Istio, You can use TSB direct mode configuration to achieve the desired result. 

* According to argoRollout convention, 2 subsets named `stable` and `canary` has been added to `subsets` with necessary labels to identify canary and stable pods which argoRollout inject based on `canaryMetadata`. 
* `istio.io/rev: tsb` label needs to be added to Gateway,VirtualService & DestinationRule objects to prevent Istiod which is running locally in ControlPlane cluster from processing it. These configurations are meant to be processed by TSB mangement plane. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: blogger-dr
  namespace: content
  labels:
    istio.io/rev: tsb
  annotations:
    tsb.tetrate.io/organization: tetrate
    tsb.tetrate.io/tenant: content
    tsb.tetrate.io/workspace: blogger-ws
    tsb.tetrate.io/trafficGroup: blogger-traffic
spec:
  host: blogger.content.svc.cluster.local
  subsets:
    - name: stable
      labels:
        app: blogger-api
        role: stable
    - name: canary
      labels:
        app: blogger-api
        role: canary
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: blogger-vs
  namespace: content
  labels:
    istio.io/rev: tsb
  annotations:
    tsb.tetrate.io/organization: tetrate
    tsb.tetrate.io/tenant: content
    tsb.tetrate.io/workspace: blogger-ws
    tsb.tetrate.io/gatewayGroup: blogger-gateway
spec:
  hosts:
    - "blogger.tetrate.com"
  gateways:
    -  blogger-gateway
  http:
    - route:
        - destination:
            host: blogger.content.svc.cluster.local
            subset: stable
          weight: 100
        - destination:
            host: blogger.content.svc.cluster.local
            subset: canary
          weight: 0
```

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: blogger-gateway
  namespace: content
  labels:
    istio.io/rev: tsb
  annotations:
    tsb.tetrate.io/organization: tetrate
    tsb.tetrate.io/tenant: content
    tsb.tetrate.io/workspace: blogger-ws
    tsb.tetrate.io/gatewayGroup: blogger-gateway
spec:
  selector:
    app: tsb-gateway-blogger
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - blogger.tetrate.com
```

Rest of the TSB configurations for `Tenant`, `Workspace`, `Groups` can be referred [here](https://github.com/sreeharikmarar/tsb-argo-rollout/blob/main/tsb/conf.yaml)

![Screenshot 2022-07-13 at 11 01 52 PM](https://user-images.githubusercontent.com/855824/178796547-4794bea2-0b95-47aa-ad82-9638702f6727.png)

### Reference Deployment from Rollout

* Create a Rollout resource and refer your existing deployment using `workloadRef`. 
* Make sure selector matchLabels has been configured based on your k8s application deployment manifest. 
* Configure `canaryMetadata` to inject labels and annotations on canary and stable pods.
* Configure Istio `virtualService` and `destinationRule` based on TSB configuration.
* Scale down your existing deployment to 0 by changing the replicas.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: rollout-example
spec:
  replicas: 4
  selector:
    matchLabels:
      app: blogger-api
      process: web
  workloadRef:
    apiVersion: apps/v1
    kind: Deployment
    name: blogger-api-web
  strategy:
    canary:
      canaryMetadata:
        annotations:
          role: canary
        labels:
          role: canary
      stableMetadata:
        annotations:
          role: stable
        labels:
          role: stable
      trafficRouting:
        istio:
          virtualService: 
            name: blogger-vs
          destinationRule:
            name: blogger-dr    
            canarySubsetName: canary  
            stableSubsetName: stable  
      steps:
      - setWeight: 5
      - pause:
          duration: 10m
```

### Trigger Canary

Update App images to canary version

```
$: kubectl argo rollouts set image rollout-example blogger-api-web=gcr.io/sreehari-test-1/blogger:0.0.9 -n content
deployment "blogger-api-web" image updated
```

### Monitor Canary Deployment

```
$: kubectl argo rollouts get rollout rollout-example --watch -n content
```

![Screenshot 2022-07-14 at 5 10 38 PM](https://user-images.githubusercontent.com/855824/178975464-1399fcc8-03ae-4144-a4e8-8eba92eccb65.png)

![Screenshot 2022-07-14 at 5 11 03 PM](https://user-images.githubusercontent.com/855824/178975655-4bc57ff9-9899-4aed-b047-5152acfcbcb5.png)

![Screenshot 2022-07-14 at 4 54 10 PM](https://user-images.githubusercontent.com/855824/178972702-899aa111-4335-46bb-b99c-3046be242bf3.png)


### Promote Canary 

```
$: kubectl argo rollouts promote rollout-example -n content
rollout 'rollout-example' promoted
```
![Screenshot 2022-07-14 at 4 57 52 PM](https://user-images.githubusercontent.com/855824/178972805-214ca3a7-7ca6-41fe-a3e3-a5cb1991f00c.png)



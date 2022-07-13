# TSB + ArgoRollout for Canary deployments

TSB mesh configurations integrated with argoCD and argoRollout for Canary deployment workflow 

### Install ArgoCD

Please refer their offical documentation [here](https://argo-cd.readthedocs.io/en/stable/getting_started/)

### Install ArgoRollout

Please refer their offical documentation [here](https://argoproj.github.io/argo-rollouts/installation/)

### Deploy Application 

* Refer application deployment spec [here](https://github.com/sreeharikmarar/tsb-argo-rollout/tree/main/application)

![Screenshot 2022-07-13 at 11 00 33 PM](https://user-images.githubusercontent.com/855824/178796152-d78b8201-82e0-4881-8c94-648f4afb2198.png)

### Enable GitOps on TSB Workload Cluster and Sync TSB configs via ArgoCD

* Refer TSB configurations [here](https://github.com/sreeharikmarar/tsb-argo-rollout/blob/main/tsb/conf.yaml)

![Screenshot 2022-07-13 at 11 01 52 PM](https://user-images.githubusercontent.com/855824/178796547-4794bea2-0b95-47aa-ad82-9638702f6727.png)

### Create Rollout and Trigger Canary

* Create Rollout, refer the config [here](https://github.com/sreeharikmarar/tsb-argo-rollout/blob/main/rollout/rollout.yaml) 

* Update App images to canary version
```
$: kubectl argo rollouts set image rollout-example blogger-api-web=gcr.io/sreehari-test-1/blogger:0.0.9 -n content
```

* Monitor Canary Deployment

```
$: kubectl argo rollouts get rollout rollout-example --watch -n content
```
![Screenshot 2022-07-13 at 9 45 16 PM](https://user-images.githubusercontent.com/855824/178797647-256a1ee1-1549-4fd9-bc66-82d183b01396.png)

![Screenshot 2022-07-13 at 10 46 11 PM](https://user-images.githubusercontent.com/855824/178797358-2e07579c-753d-4b01-8b9d-b37f57db83a5.png)


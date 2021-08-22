# Step by step notes

## Set up the cluster

Use the eksctl utility and save yourself some time experimenting.
I spent many hours trying to set up a usable cluster using the awscli and web-ui - it is not worth the headache.
This took around 22 minutes:

```bash
eksctl create cluster --name udagramCluster --region=us-central-1 --nodes-min=2 --nodes-max=3
```

Validate the connection & available nodes

```bash
kubectl cluster-info
kubectl get nodes
```

## Apply config files

```bash
kubectl apply -f aws-secret.yaml
kubectl apply -f env-configmap.yaml
kubectl apply -f env-secret.yaml
```

Double check the AWS credentials, wrong credentials may look like CORS errors when accessing your s3 bucket.
Create your credential-base64-string using:

```bash
cat ~/.aws/credentials | head -n 3
cat ~/.aws/credentials | head -n 3 | base64
```

Use the output in your aws-secret yaml without line breaks:

```yaml
(...)
data:
  credentials: W2Rl(...)
```

## Deploy services

```bash
kubectl apply -f udagram-api-feed-deployment.yaml
kubectl apply -f udagram-api-feed-service.yaml
kubectl apply -f udagram-api-user-deployment.yaml
kubectl apply -f udagram-api-user-service.yaml
kubectl apply -f udagram-reverseproxy-deployment.yaml
kubectl apply -f udagram-reverseproxy-service.yaml
kubectl apply -f udagram-frontend-deployment.yaml
kubectl apply -f udagram-frontend-service.yaml
```

Validate the services & check errors:

```bash
kubectl get pod
kubectl get pod -w  [ -> This allows you to watch as the status changes ]
kubectl describe pod udagram-api-feed-bd9b74497-hrxgl
kubectl logs udagram-api-feed-bd9b74497-hrxgl
kubectl logs udagram-api-feed-bd9b74497-hrxgl -f [ -> This allows you to follow as logs are added]
```

## Remember to update the frontend service

Get your frontend and reverseproxy external url using:

```bash
kubectl get svc
```

The relevant field is "External-IP". Also note that this shows the currently used ports - validate them!

Update `udagram-frontend/src/environments/environment.prod.ts` and `environment.ts` with the reverseproxy service url.
Make sure you use the reverseproxy service external url here! Also double check the ports (config, deployment, service)!

Next, set the url variable in the `env-configmap.yaml`.
Make sure you use the frontend service external url here! Also double check the ports (config, deployment, service)!

Now update all pods/deployments to make sure they use the new url environment variable.
Be careful when updating/deleting/scaling your services though, as re-deploying the services of the frontend or revserproxy changes their external url!

## Metrics & HPA / Autoscale

Installing the metrics component is described [here](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html).

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl get deployment metrics-server -n kube-system
```

Check the result, it should return a single metrics-server after a few seconds.

```bash
kubectl get deployment metrics-server -n kube-system
```

Now add autoscaling to your pods using

```bash
kubectl autoscale deploy udagram-frontend --min=2 --max=5 --cpu-percent=80
```

Now `kubectl get hpa` should give you the desired results.

## Helpers

### Update the containers

```bash
kubectl scale deploy <your-deployment> --replicas=0
kubectl scale deploy <your-deployment> --replicas=2
```

or

```bash
kubectl delete -f udagram-api-feed-deployment.yaml
kubectl apply -f udagram-api-feed-deployment.yaml
```

### Enter the commandshell of a container

```bash
kubectl exec --stdin --tty udagram-api-feed-bd9b74497-hgwvg -- /bin/sh
kubectl run -i --tty --rm debug --image=busybox --restart=Never -- sh
```

# Kubernetes Deploy Example

## DEPLOY
We will use this yml file to create a deployment to the kubernetes cluster.   
***deployment.yml***
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```


#### Lets look at the current deployments

```bash
% kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
```


We can deploy a base nginx image to the cluster in this example. This will service but an empty nginx web server
that we can curl to later.
Deploy the service to the kubernetes cluster with this command
```bash
% kubectl apply -f example/deployment.yml
```
Now lets look at the deployments
```bash
% kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx   2/2     2            2           19h
```

The image is now deployed but is not available outside of the kubernetes cluster until we Expose it with a 
service.


## SERVICE
We will use this yml file to create a service to expose the deployment in kubernetes cluster to an external device.   
***service.yml***
```yml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  type: LoadBalancer
  selector:
    run: my-nginx
```
Run the following command to install the service on the cluster
```bash
% kubectl apply -f example/service.yml
```


```bash
$ kubectl get services
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP        5d
my-nginx     LoadBalancer   10.96.94.141   localhost     80:32671/TCP   12m << shows service available to locahost
```

Now your can curl to the service on your localhost

```bash
$ curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

$

```

You can verify the service is available to any other hosts on the same network with a curl those hosts

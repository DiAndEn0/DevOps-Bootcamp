
After installing VM-Tools using the terminal and verifiying they're working, installing Docker for the Ubuntu VM:
- Using this [site](https://docs.docker.com/engine/install/ubuntu/) and this [one](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
- After confirming docker is running using `https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository` 
  I'm trying to verify if the installation is successful by runnign the `hello-world` image
	- `sudo docker run hello-world`
	- The installation was successful, continuing working. ![[Pasted image 20260517144215.png]]
With the installation done, I've downloaded the necessary libraries for using `docker` and `vim` and started working on the docker fundamentals.

- Using `docker pull quay.io/brancz/prometheus-example-app:v0.3.0` I've pulled the image of a prometheus-example app that exposes its metrics via `/metrics` endpoint.
- Using `docker pull  prom/prometheus:latest` I've pulled the **Prometheus** image I'll be using for this practical part of the bootcamp.
- I've also pulled `nginx` image and the `alpine` image to help me with tests and see if my configs are working correctly.

By running `docker images` I'm able to see all of my current images. There are a few more from other tests but we will focus only on 4:
![[Pasted image 20260521213545.png]]

Our main images are `alpine:latest`, `nginx:latest`, `prom/prometheus:latest`, `quay.io/brancz/prometheus-example-app:v0.3.0`

- Before running any of the containers, I will first create a new **bridge network** which we will use to isolate our containers and also create an environment where they could communicate with each other.

- By using this command I've created a new `my-network` network which would suffice for our current objective:
```
docker network create my-network
```
- After that I will create a volume called `prom-volume` which will be bind to a directory where we would create our `prometheus.yml` file and then mount it onto our `prometheus` container.
	`docker volume create prom-volume`

- Before running any containers, I will first configure my `prometheus.yml` file which we would be using to run our **Prometheus** container.
	`cd` - to go to the main linux directory
	`cd /etc` - go into the config directory 
	`sudo mkdir prometheus/` - to create a directory for prometheus
	`cd prometheus` - go to the newly created directory
	`sudo vim prometheus.yml` - create a new yml file using `vim`
- The configuration is as follows:
	![[Pasted image 20260522000855.png]]
	We set global configs for all jobs, and then we create a job named `node-prometheus_example`, with the target being our example app's NAT Ip that it would get once we connect its container to `my-network`


- Now we will start running our containers, we will first run:
```
``docker container run -dit -p 8080:8080  --name example-app-cont quay.io/brancz/prometheus-example-app:v0.3.0`
```
 - starting our example app and giving it a name

```
`docker container run -dit --name alpine-cont alpine:latest`
```
 - running our lightweight alpine container

```
`docker container run -dit --name nginx-c1 nginx:latest`
```
 - to run our nginx web page

- After that we use a combination of `docker network disconnect` and `docker network connect` to disconnect our newly made containers from the default network and connect them to our custom one.
- With that we can check what is the IP given to our example app inside of the network by writing `docker network inspect my-network`
- After finding the IP, check if the `prometheus.yml` has the correct IP, if not go back and change it to the correct one.
- Now we can finally run our last container which will be **Prometheus** itself.

```
`docker run -dit  --name mounted_prometheus  -p 9090:9090     -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro  -v prom-volume:/etc/prometheus     prom/prometheus ` 
```

-  we bind the config file to our config file and then mount our volume on top of the `/etc/prometheus` directory we just created, meaning we don't have to create new containers when we configure the `yml` file anew.



After running everything this is how it should like after running `docker ps`:
![[Pasted image 20260522002954.png]]


Now by going to `http://172.18.0.5:9090/targets` we can go to the **Prometheus** interface and observe if we did everything correctly:
![[Screenshot 2026-05-22 003152.png]]
Since it is up, we have established a correct connection and `Prometheus` is successfully scraping the data.


### K3S

I've installed K3S as a Service using this command:
````
curl -sfL https://get.k3s.io | sh -
````

tutorial:
https://www.youtube.com/watch?v=qZpeiZGJ98A

Swap off:
```
sudo swapoff -a

sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Pointing kubectl to k3s config file
```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```


Label the node as 'master':
```
kubectl label node ubuntuvm node-role.kubernetes.io/master=master
```

I've made a new pod to test out the features of k3s and kubectl:
![[Pasted image 20260602120930.png]]
I've applied the file using `kubectl apply -f` which created the pod from the `.yaml` file
```
vim prom-example-scraper.yaml

kubectl apply -f prom-example-scraper.yaml
```

[Prometheus monitoring on kubernetes](https://www.youtube.com/watch?v=QoDqxm7ybLc)

In order to create the environment needed for the deployment of Prometheus and scraping of the example application, I first had to establish the namespace for the monitoring in order to make the system modular and differentiate between deployments:

```
kubectl create namespace monitoring
```

I've made  a path to my new `kustomize` folder in which I would use a `kustomization.yaml` file to apply all of my `.yaml` files together. 

In the `/k3s-example/base` folder I've created the following files:
- `cluster-role-prom.yaml` - a `ServiceAccount` file which would allow prometheus to have the security policy to scrape and find endpoints in our VM cluster for metrics![[Pasted image 20260602122054.png]]
- `cluster-role-prom-scrape.yaml` - a `ClusterRole` file describing prometheus' role in the cluster and what it's allowed to do (which resources to scrape, with which commands)![[Pasted image 20260602122440.png]]
- `config-prom.yaml` - a `ConfigMap` file detailing prometheus' configuration file, meaning its scraping behavior and the different configs. This specific config file uses an automatic discovery based on annotations of endpoints provided by the Kubernetes API. ![[Pasted image 20260602122648.png|675]]
- `role-binding-prom.yaml` - a `ClusterRoleBinding` file which binds the `ClusterRole` to the `ServiceAccount` we created earlier, meaning if a pod has this `ServiceAccount` declared in its variables it would have all of the roles provided by the `ClusterRole`. ![[Pasted image 20260602123351.png]]
- `service-prom.yaml` - a `Service` which would expose prometheus to the cluster on an endpoint we provide. The type of service is `NodePort` as we want to port-forward it so we can access it via browser or other external means on our VM. 
  ![[Pasted image 20260602124032.png]]
- `service-prom-app.yaml` - another `Service` four the example app, it exposes its endpont with a NodePort for testing purposes and also uses custom annotations for Prometheus scraping discovery.
  ![[Pasted image 20260602124253.png]]
- `deployment-prom.yaml` - our `Deployment` of Prometheus. In order to bind our custom config file we mount it with a volume, while also declaring the pod to have our custom `ServiceAccount` for the role permissions.
  ![[Pasted image 20260602125050.png]]
- `deployment-prom-sample-app.yaml` - `Deployment` file of the example app, nothing that difficult, basic configurations for a one pod deployment.![[Pasted image 20260602125338.png]]
- `deployment-alpine-ping.yaml` - `Deployment` file of our http request senders. We use `replicaSets`  to make 3 identical pods with the startup command to ping the service that exposes prom-example app every 5 seconds. We can use the service name directly as they exist within the same namespace.![[Pasted image 20260602130204.png]]
-  And finally `kustomization.yaml` - the `kustomize` file that we use to bind and run everything together. We provide all of the resources to generate the corresponding objects from them, and we use `configMapGenerator` to take our `config-prom.yaml` and generate a `ConfigMap` which we will bind to the `monitoring` namespace and use it as the `prometheus.yml` config file.
  ![[Pasted image 20260602130544.png]]

After everything's done, we can use this command to apply everything:
```
kubectl apply -k . # use this in our /base directory where all of the files are
```

Using K9s we can look at our namespaces and see everything working:
 ![[Pasted image 20260602131148.png]]
![[Pasted image 20260602131209.png]]

On the Prometheus page we can see our metrics available:
![[Pasted image 20260602131241.png]]

This is used for port-forwarding manually:
 ```
kubectl port-forward deployment/prometheus 9090
```  
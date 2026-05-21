
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
	`docker network create my-network`-

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
	`docker container run -dit -p 8080:8080  --name example-app-cont quay.io/brancz/prometheus-example-app:v0.3.0
	 - starting our example app and giving it a name
	
	`docker container run -dit --name alpine-cont alpine:latest` - running our lightweight alpine container
	
	`docker container run -dit --name nginx-c1 nginx:latest` - to run our nginx web page
	
- After that we use a combination of `docker network disconnect` and `docker network connect` to disconnect our newly made containers from the default network and connect them to our custom one.
- With that we can check what is the IP given to our example app inside of the network by writing `docker network inspect my-network`
- After finding the IP, check if the `prometheus.yml` has the correct IP, if not go back and change it to the correct one.
- Now we can finally run our last container which will be **Prometheus** itself.
	
	`docker run -dit  --name mounted_prometheus  -p 9090:9090     -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro  -v prom-volume:/etc/prometheus     prom/prometheus ` 
	
	-  we bind the config file to our config file and then mount our volume on top of the `/etc/prometheus` directory we just created, meaning we don't have to create new containers when we configure the `yml` file anew.
	


After running everything this is how it should like after running `docker ps`:
![[Pasted image 20260522002954.png]]


Now by going to `http://172.18.0.5:9090/targets` we can go to the **Prometheus** interface and observe if we did everything correctly:
![[Screenshot 2026-05-22 003152.png]]
Since it is up, we have established a correct connection and `Prometheus` is successfully scraping the data.
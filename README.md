
## Project Overview

The purpose of this project is to detail the steps of deploying the Fossa all-in-one appliance in single node Kubernetes cluster.

The following depicts the topology of the Fossa all in one appliance.


---

###  System Requirements
-   At least 16 GB 
- 	400 GB in / volume
- 	Docker installed
- 	Kubernetes single node cluster
- 	TLS certificate for 
-  - Fossa application host
-  - Minio application host



## Setup Script steps

* run_docker.sh
    1. This script will build a docker container with the included docker file
    2. list the docker images
    3. run the docker container
    4. expose the container on the host on port 8080
    
*  upload_docker.sh
    1. tag the image for docker repo
    2. login into docker hub
    3. push image inti docker hub


*  run_kubernetes.sh
    1. deploy the container as a kubernetes deployment
    2. list kubernetes pods
    3. expose the deployment on port 8080



### Running `app.py`

1. Standalone:  `python app.py`
2. Run in Docker:  `./run_docker.sh`
3. Run in Kubernetes: ./run_kubernetes.sh`

### Kubernetes Steps

* Setup and Configure Docker locally
* Setup and Configure Kubernetes locally
* Create Flask app in Container
* Run via kubectl


## Explaination of file

* docker_out.txt (log output for docker container)
* kubernetes_out.txt (output of run_kubernetes.sh script)
* kubernetes.out.txt (output of make_predictions.sh on kubernetes deployment)



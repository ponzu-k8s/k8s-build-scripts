ponzu-k8s build files

# howto deploy ponzu-k8s on gke

for https://github.com/ponzu-k8s

## setup new gke project
* go to https://console.cloud.google.com/start & create a new project named ‘ponzu-example’ (dont put quotes in name)
* enable billing for project at https://console.cloud.google.com/billing
* enable compute engine api at https://console.cloud.google.com/apis/api/compute_component/overview
* enable container engine api at https://console.cloud.google.com/apis/api/container/overview

## setup gke toolchain
* install google cloud sdk & kubectl
  - install locally to interact with gke from your local console:
    * https://cloud.google.com/sdk/docs/quickstarts
    * (in console) $ gcloud components install kubectl
  - *or* use the google cloud shell: https://cloud.google.com/shell/docs/quickstart
* set defaults for gcloud (in console)
  - `$ gcloud auth login`
  - `$ gcloud config set project ponzu-example`
  - `$ gcloud config set compute/zone us-central1-b`
  
## create gke cluster (costs $/hour)
* create 1-node cluster (in console)
- `$ gcloud container clusters create ponzu-cluster \` \
`--disk-size 100 \` \
`--machine-type n1-standard-1 \` \
`--num-nodes 1`
* create persistent disks
  - `$ gcloud compute disks create --size 10GB admin-disk`
    (admin-disk holds ponzu project files, contributed content & uploads)
  - `$ gcloud compute disks create --size 1GB web-disk`
    (web-disk holds html & javascript files for nginx frontend)

## mount & format persistent disks
* find vm instance name that matches gke-ponzu-cluster-default-pool-<*>-<*>
  - `$ gcloud compute instances list`
    * prints out list of gce vm instances
    * find instance with ‘ponzu-cluster’ in the name
    * in following command replace gke-ponzu-cluster-... with that name
  - `$ export INSTANCE_NAME=gke-ponzu-cluster-default-pool-<*>-<*>`
    * for example: $ export INSTANCE_NAME=gke-ponzu-cluster-default-pool-e3d8f4f4-jn4h
* attach disks to instance (this is only necessary to initialize disks, k8s will later automatically attach admin-disk and web-disk to the proper containers)
  - `$ gcloud compute instances attach-disk $INSTANCE_NAME --disk admin-disk`
  - `$ gcloud compute instances attach-disk $INSTANCE_NAME --disk web-disk`
* ssh into instance 
  - `$ gcloud compute ssh $INSTANCE_NAME`
* list block disks (~$ is remote console prompt)
  - `~$ lsblk`
    * should print out a list of disks & partitions with the following two lines at the end:
      ```
      sdb       8:16   0   10G  0 disk
      sdc       8:32   0    1G  0 disk
      ```
    * if names sd* are different change them in commands below
* format 10G admin-disk on /dev/sdb and mount at /mnt/disks/admin
  - `~$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb`
  - `~$ sudo mkdir -p /mnt/disks/admin`
  - `~$ sudo mount -o discard,defaults /dev/sdb /mnt/disks/admin`
  - `~$ sudo chmod a+w /mnt/disks/admin`
* format 1G web-disk on /dev/sdc and mount at /mnt/disks/web
  - `~$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdc`
  - `~$ sudo mkdir -p /mnt/disks/web`
  - `~$ sudo mount -o discard,defaults /dev/sdc /mnt/disks/web`
  - `~$ sudo chmod a+w /mnt/disks/web`
* list block disks again to check that admin & web partitions are now mounted
  - `~$ lsblk`
    * last two lines should now look something like this:
      ```
      sdb       8:16   0   10G  0 disk /mnt/disks/admin
      sdc       8:32   0    1G  0 disk /mnt/disks/web
      ```

## pull initial persistent disk files from git repo
* pull admin disk from <https://github.com/ponzu-k8s/example-admin-disk.git>
  - `~$ git clone https://github.com/ponzu-k8s/example-admin-disk.git /mnt/disks/admin`
* pull web disk from <https://github.com/ponzu-k8s/example-web-disk.git>
  - `~$ git clone https://github.com/ponzu-k8s/example-web-disk.git /mnt/disks/web`

## (or upload local files)
* change dir to ponzu-k8s folder
  - `$ cd ponzu-k8s`
* use gclouds scp tool to copy admin & web dirs to persistent disks mounted to vm
  - `$ gcloud compute scp --recurse ./admin ${INSTANCE_NAME}:/mnt/disks`
  - `$ gcloud compute scp --recurse ./web ${INSTANCE_NAME}:/mnt/disks`

## *unmount & detach disks*
**(if you skip this step persistent disks will not be available for k8s deployment!)**

* (ssh into instance again if you disconnected)
$ gcloud compute ssh $INSTANCE_NAME
* unmount disks
  - `~$ sudo umount /mnt/disks/admin`
  - `~$ sudo umount /mnt/disks/web`
* exit ssh
  - `~$ exit`
* detach persistent disks from vm instance
  - `$ gcloud compute instances detach-disk $INSTANCE_NAME --disk admin-disk`
  - `$ gcloud compute instances detach-disk $INSTANCE_NAME --disk web-disk`

## pull docker & k8s build files from git repo
* clone github.com/ponzu-k8s/build repo locally
  - `$ git clone https://github.com/ponzu-k8s/build.git ~/ponzu-k8s/build`

## build & upload docker images
* install docker on your system -> <http://www.letmegooglethat.com/?q=install+docker>
* enable google container registry api for project at <https://console.cloud.google.com/apis/api/containerregistry.googleapis.com/overview>
* start docker
  - `$ sudo systemctl start docker`
* build & upload admin container image
  - `$ cd ~/ponzu-k8s/build/docker/admin`
  - `$ docker build -t us.gcr.io/ponzu-example/admin:v1 .`
  - `$ gcloud docker -- push us.gcr.io/ponzu-example/admin:v1`
* (run locally)
  - `$ git clone https://github.com/ponzu-k8s/example-admin-disk.git ~/ponzu-k8s/example-admin-disk/`
  - `$ cd ~/ponzu-k8s/example-admin-disk/`
  - `$ docker run -v $(pwd)/:/go/src/project -p 8080:8080 -it us.gcr.io/ponzu-example/admin:v1 /bin/bash`
  - go to <http://localhost:8080/admin> in a browser
* build & upload web container image
  - `$ cd ~/ponzu-k8s/build/docker/web`
  - `$ docker build -t us.gcr.io/ponzu-example/web:v1 .`
  - `$ gcloud docker -- push us.gcr.io/ponzu-example/web:v1`

## start deployment in gke
* change to directory with ponzu-k8s yaml files
  - `$ cd ponzu-k8s/yaml`
* use kubectl to start admin deployment 
  - `$ kubectl create -f admin-deployment.yaml`
* check that admin container started successfully
  - `$ kubectl get pods`
    * should show admin-<*> pod ready & running
* start admin service
  - `$ kubectl create -f admin-service.yaml`
* check that admin service is available on k8s cluster on port 8080
  - `$ kubectl get services`
    * should list admin service on port 8080 & kubernetes service on port 443
* start web deployment
  - `$ kubectl create -f web-deployment.yaml`
* check that web container started successfully
  - `$ kubectl get pods`
    * should show both admin-<*> and web-<*> pods ready & running
* start web service & web ingress to allow traffic from internet into k8s cluster
  - `$ kubectl create -f web-service.yaml`
  - `$ kubectl create -f web-ingress.yaml`
* check status of ingress controller
  - `$ kubectl describe ingress web-ingress`
    * will print out a bunch of info including allocated ip address
    * under ‘annotations’ status of backend will initially be “unknown”
    * after ~10 mins status will change to “healthy” or “unhealthy”
    * once status is healthy you should be able to visit ponzu example by pasting the ingress ip address into your browser
    * alternately check ingress status in google cloud console here -> <https://console.cloud.google.com/net-services/loadbalancing/loadBalancers/list>


## ssh into running gke container
* find name of pod
  - `$ kubectl get pods`
  - (find pod name)
  - `$ export POD_NAME=<pod-name>`
* use pod name to execute a bash shell on the pod thru ssh
  - `$ kubectl exec -it $POD_NAME -- /bin/bash`

## (references)
* deis k8s howto at <https://deis.com/blog/2016/first-kubernetes-cluster-gke/>
* gke quickstart at <https://cloud.google.com/container-engine/docs/quickstart>
* gce persistent disk howto -> <https://cloud.google.com/compute/docs/disks/add-persistent-disk>
* gce ssh into instance howto -> <https://cloud.google.com/compute/docs/instances/connecting-to-instance>
* gce scp docs -> <https://cloud.google.com/sdk/gcloud/reference/compute/scp>
* k8s ingress howto -> <https://medium.com/@cashisclay/kubernetes-ingress-82aa960f658e>

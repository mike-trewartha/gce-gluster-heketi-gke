# Gluster & Heketi on GCE providing dynamic storage provisioning for Kubernetes in GKE... Oh My!
Yes, the title is a mouthful... deal with it!

[![N|Solid](https://code.google.com/codejam/contest/static/powered_by_gcp_logo.png)](https://cloud.google.com/)

### Introduction
This guide will walk you through the process of deploying a [Gluster](https://www.gluster.org/) cluster using [CentOS7](https://www.centos.org/) built on [GCE](https://cloud.google.com/compute/), with [Heketi](https://github.com/heketi/heketi) providing an API layer on top allowing [Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#glusterfs) on [GKE](https://cloud.google.com/container-engine/) to dynamically provision storage when new deployments are created.

Why? What if you needed a persistent storage layer across your GKE environment, which has been provisioned with 3 GKE nodes in different zones to provide fault tolerance. What if you're trying to deploy stateful applications?

Most deployments of Gluster and Heketi provision the Gluster storage nodes as pods within a Kubernetes cluster, however when opting for a managed Kubernetes service as provided in the Google Cloud Platform by Google Container Engine (GKE) thing's aren't quite as simple. Heketi only supports the [XFS](https://en.wikipedia.org/wiki/XFS) file system, Google however, have deprecated their [Container-VM](https://cloud.google.com/compute/docs/containers/container_vms) OS which does support XFS (and was based on Debian7) -  it is due to be removed as an options for GKE cluster deployments in September 2017! Google's replacement GKE Node OS, and their default and reccomended choice is [COS (Container-Optimised OS)](https://cloud.google.com/container-optimized-os/)  on the other hand only supports [EXT4](https://en.wikipedia.org/wiki/Ext4). Never fear, there is a simple solution! Why not provisiong a Gluster cluster on Google Compute Engine, and point your Google Container Engine there for storage?

### Assumptions!
I've attempted to write this guide so that readers of any technical skill can follow, however I do make a few assumptions:
  - You have basic Linux skills.
  - You have an understanding of GKE, Kubernetes, Gluster and Heketi, at least what they are and what they do.
  - You have a GCP account created, and a new project (heketi) ready.
  - You have the [gcloud](https://cloud.google.com/sdk/gcloud/) command line tools installed locally.
  - You have the [heketi-cli](https://github.com/heketi/heketi/tree/master/client/cli/go) installed.
  - Heketi will be installed on one of the Gluster nodes. This guide will be updated to move this to a pod on GKE at a later date.
  - Optional: You have [git](https://git-scm.com/) installed - if you wish to clone my [github repo](https://github.com/mike-trewartha/).


Also I assume that you:
  - Are using the [australia-southeast1](https://cloud.google.com/about/locations/sydney/) GCP region.
  - You're happy with a 30GB Gluster Volume for  (due to the SSD limitations of the ["Free" GCP  account](https://cloud.google.com/free/).
  - You want 1 GCE Instance provisioned in each of the 3 zones in the australia-southeast1 region.
  - Each compute instance will be named "gluster1-a", "gluster1-b" and "gluster1-c", taking the zone into the name.
  - Each compute instance attached storage disk will have a name to indicate the zone and instance it is associated with, aka: "gluster1-a-disk-1".
  - Gluster will use replicated volumes, meaning 3x 30GB disks across 3 GCE nodes will yield in 30GB of Gluster storage (meaning a write to any disk will be replicated to the other 2 disks)

Hopefully I haven't scared anyone away. Let's dig in!


### Prepare GCP Infrastructure

First up, because GKE's Kubernetes master lives in a different project from the GKE nodes, it doesn't have the freedom to communicate to the GCE instances where we will be installing Heketi or Gluster. It can, however, access the public internet! So create a firewall rule that allows port 8080 access from the public internet *this should probably be locked down a bit more before pushing this into production* to our tag 'heketi'
```sh
gcloud beta compute --project "heketi" firewall-rules create "heketi" --allow tcp:8080 --direction "INGRESS" --priority "1000" --network "default" --source-ranges "0.0.0.0/0" --target-tags "heketi"
```

Next, it's probably wise to build a GKE cluster!
```sh
gcloud container --project "heketi" clusters create "cluster-1" --zone "australia-southeast1-a" --cluster-version "1.7.0" --machine-type "n1-standard-1" --image-type "COS" --disk-size "100" --scopes "https://www.googleapis.com/auth/compute","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --network "default" --enable-cloud-logging --enable-cloud-monitoring --additional-zones "australia-southeast1-b","australia-southeast1-c" --enable-legacy-authorization
```

Great, next we'll prepare some GCE SSD Persistent Disks to become the Gluster bricks
```sh
for i in a b c; do gcloud compute disks create "gluster1-$i-disk-1" --size "30" --zone "australia-southeast1-$i" --type "pd-ssd"; done
```

Time to deploy our 3 GCE instances which will run our Gluster cluster.
```sh
for i in a b c; do gcloud compute --project "heketi" instances create "gluster1-$i" --zone "australia-southeast1-$i" --machine-type "n1-standard-1" --subnet "default"  --maintenance-policy "MIGRATE" --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --image "centos-7-v20170620" --image-project "centos-cloud" --boot-disk-size "10" --boot-disk-type "pd-standard" --boot-disk-device-name "gluster1-$i"; done
```

Finally, attach the SSD PD's to the GCE instances
```sh
gcloud compute instances attach-disk gluster1-a --disk=gluster1-a-disk-1 --zone australia-southeast1-a
gcloud compute instances attach-disk gluster1-b --disk=gluster1-b-disk-1 --zone australia-southeast1-b
gcloud compute instances attach-disk gluster1-c --disk=gluster1-c-disk-1 --zone australia-southeast1-c
```

Infrastructure Check! By now you should have an inbound firewall ACL, a 3 node GKE cluster, 3 GCE instances each with a SSD-PD attached.

### Software Configuration
Let's install the Gluster packages on the GCE instances. I had to split this into 2 yum install commands, I found that often packages weren't being installed correctly. Maybe PEBKAC, maybe not, let me know!
```sh
for i in a b c; do gcloud compute ssh --zone australia-southeast1-${i} gluster1-${i} --command "sudo yum install -y centos-release-gluster310 && sudo yum install -y glusterfs gluster-cli glusterfs-libs glusterfs-server lvm2"; done
```

Next you'll need to log into each GCE Instance manually, and su - to root. There is an know bug with systemd and sudo which prevents non-root users from using sudo to stop/start services. Also, you probably don't need to disable SELINUX, however as a precaution, lets just go ahead and do that. Finally, we'll need to enable key based authentication for the root users, as Heketi uses SSH to run commands against Gluster.
```sh
gcloud compute ssh --zone australia-southeast1-a gluster1-a
sudo su -
setenforce 0 && systemctl enable --now glusterd.service  && sed -i 's/PermitRootLogin no/PermitRootLogin without-password/g' /etc/ssh/sshd_config && systemctl restart sshd.service
```
```sh
gcloud compute ssh --zone australia-southeast1-b gluster1-b
sudo su -
setenforce 0 && systemctl enable --now glusterd.service  && sed -i 's/PermitRootLogin no/PermitRootLogin without-password/g' /etc/ssh/sshd_config && systemctl restart sshd.service
```
```sh
gcloud compute ssh --zone australia-southeast1-c gluster1-c
sudo su -
setenforce 0 && systemctl enable --now glusterd.service  && sed -i 's/PermitRootLogin no/PermitRootLogin without-password/g' /etc/ssh/sshd_config && systemctl restart sshd.service
```

We need to tell one of the Gluster nodes to go make friends with the other nodes, cause they're notorious loners otherwise.
```sh
gcloud compute ssh --zone australia-southeast1-a gluster1-a --command "sudo gluster peer probe gluster1-b && sudo gluster peer probe gluster1-c && sudo gluster peer status"
```

Time to install Heketi on the gluster1-a GCE instance.
```sh
gcloud compute ssh --zone australia-southeast1-a gluster1-a --command "sudo yum -y install heketi"
```

Better jump on gluster1-a and set up the ssh key pair
```sh
gcloud compute ssh --zone australia-southeast1-a gluster1-a
sudo su -
ssh-keygen -f /etc/heketi/heketi_key -t rsa -N '' && chown heketi:heketi /etc/heketi/heketi_key* && echo "---" && cat /etc/heketi/heketi_key.pub
```

Back on your local machine, grab the output of the public key from above, and replace it in the command below (ie: '<<PUBLIC_KEY>>' should become 'ssh-rsa AAAAB3NzaC...lnst root@gluster1-a' )
```sh
for i in a b c; do gcloud compute ssh --zone australia-southeast1-${i} gluster1-${i} --command "umask 0077 && sudo mkdir /root/.ssh/ && echo '<<PUBLIC_KEY>>' | sudo tee --append /root/.ssh/authorized_keys"; done
```

Before we forget, lets tag that gluster1-a GCE instance to allow that firewall rule we created at the beginning to permit traffic to TCP:8080 from the public internet!
```sh
gcloud compute instances add-tags gluster1-a --tags heketi --zone australia-southeast1-a
```

Jump back into gluster1-a and finalise the Heketi configuration
```sh
gcloud compute ssh --zone australia-southeast1-a gluster1-a
sudo su -
cp /etc/heketi/heketi.json /etc/heketi/heketi.json.bak
sed -i 's/path\/to\/private_key/\/etc\/heketi\/heketi_key/g' /etc/heketi/heketi.json
sed -i 's/sshuser/root/g' /etc/heketi/heketi.json
sed -i 's/Optional: ssh port.  Default is 22/22/g' /etc/heketi/heketi.json
sed -i 's/Optional: Specify fstab file on node.  Default is \/etc\/fstab/\/etc\/fstab/g' /etc/heketi/heketi.json
sed -i 's/"executor": "mock",/"executor": "ssh",/g' /etc/heketi/heketi.json
sed -i 's/"My Secret"/"secretk3y!"/g' /etc/heketi/heketi.json
systemctl enable heketi
systemctl restart heketi
```

Test that heketi is working, look for the result "Hello from Heketi" from the following curl command.
```sh
gcloud compute ssh --zone australia-southeast1-a gluster1-a
curl gluster1-a:8080/hello
```

Finally, push out the topology of your Gluster environment to Heketi, using my included topology.json file you'll have 3 GCE nodes, in 3 zones, each with an unformatted disk located at /dev/sdb. This will need to be done where your heketi-cli client is installed. The IP address in the environment variable is the public IP address of the GCE instance where heketi is installed, the one we tagged ealier.
```sh
export HEKETI_CLI_SERVER=http://35.189.14.88:8080
/Users/michaelt/Downloads/heketi-client/bin/heketi-cli topology load --json=topology.json
Creating cluster ... ID: ee3163e9faeff6d57a6b515ae46bb045
	Creating node gluster1-a ... ID: a9561680d6411fada6d8b49838afd4b6
		Adding device /dev/sdb ... OK
	Creating node gluster1-b ... ID: 43ac26d4e1b3baf08a57774599981aa8
		Adding device /dev/sdb ... OK
	Creating node gluster1-c ... ID: 4c4ac832ac63cad921fadcfca6093ff2
		Adding device /dev/sdb ... OK
```

Software Check! Hopefully by now, you've installed Gluster and configured a 3 node Gluster cluster. You should have also configured Heketi for SSH execution, and configured Heketi with the details of your Gluster cluster!


### Kubernetes Time!
Final step! Lets run through the process of deploying a [WordPress](https://wordpress.org/) environment on our newly minted GKE cluster, backed by our shiny highly available Gluster storage platform living on GCE!

First you'll need to add the Heketi key to a Kubernetes secret, and create a storage class which uses this secret key to authenticate against Heketi. Update the gluster-sc.yaml file as required
```sh
cat gluster-sc.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: default
data:
  # base64 encoded password. E.g.: echo -n "secretk3y!" | base64
  key: c2VjcmV0azN5IQ==
type: kubernetes.io/glusterfs
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glustersc
provisioner: kubernetes.io/glusterfs
parameters:
  # The IP address of your CGE Instance which is running Heketi
  resturl: "http://35.189.14.88:8080"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"


kubectl create -f gluster-sc.yaml


kubectl describe sc glustersc
Name:		glustersc
IsDefaultClass:	No
Annotations:	<none>
Provisioner:	kubernetes.io/glusterfs
Parameters:	restauthenabled=true,resturl=http://35.189.14.88:8080,restuser=admin,secretName=heketi-secret,secretNamespace=default
Events:		<none>
```


Next we'll define a MySQL root password in a password.txt file, and add it to Kubernetes as a secret. *If you're using OSX, replace the --delete with a -d!!!* Superior BSD based systems don't run filthy GNU applications, and flags are often a little different.
```sh
tr --delete '\n' <password.txt >.strippedpassword.txt && mv .strippedpassword.txt password.txt
kubectl create secret generic mysql-pass --from-file=password.txt

kubectl describe secret mysql-pass
Name:		mysql-pass
Namespace:	default
Labels:		<none>
Annotations:	<none>

Type:	Opaque

Data
====
password.txt:	6 bytes
```


Now for the real fun, lets create our first pod, the mysql instance for our WordPress deployment!
```sh
kubectl create -f mysql-deployment.yaml

kubectl get pods
NAME                               READY     STATUS        RESTARTS   AGE
wordpress-mysql-3680404990-7pdd8   1/1       Running       0          12s
```

Time to bring our WordPress pod online. This deployment contains a LoadBalancer service, which will also provision a GCP LoadBalancer and point it at our GCP WordPress pod.
```sh
kubectl create -f wordpress-deployment.yaml

kubectl get pods
NAME                               READY     STATUS        RESTARTS   AGE
wordpress-1323938640-21xvc         1/1       Running       0          2m
wordpress-mysql-3680404990-7pdd8   1/1       Running       0          5m
```

Finally list your Kubernetes services to find the External IP of your wordpress pod. If you can brows to this IP, you'll find a WordPress installation waiting for you. Well done, I'm proud.

```sh
kubectl get svc
NAME                                 CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
glusterfs-dynamic-mysql-glusterpvc   10.51.244.243   <none>          1/TCP          47m
glusterfs-dynamic-wp-glusterpvc      10.51.253.209   <none>          1/TCP          32m
kubernetes                           10.51.240.1     <none>          443/TCP        1d
wordpress                            10.51.253.17    35.189.31.205   80:32451/TCP   47m
wordpress-mysql                      None            <none>          3306/TCP       32m
```

Hopefully that worked! Let me know if you have any questions or queries!


### Reference
I found the following references very helpful in completing the above. In no particular order:
- https://wiki.centos.org/HowTos/GlusterFSonCentOS
- https://keithtenzer.com/2017/03/24/storage-for-containers-using-gluster-part-ii/
- https://github.com/gluster/gluster-kubernetes/tree/master/docs/examples/hello_world
- https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/
- https://www.youtube.com/playlist?list=PLj_IGCS9P2Sn_MwgDO5gMXPGa88h7cPN2
- kubernetes.slack.com
- googlecloud-community.slack.com

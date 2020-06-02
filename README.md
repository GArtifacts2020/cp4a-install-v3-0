# Installation of IBM Cloud Pak for Applications v3.0 from Partner World

Given below the steps to install the Cloud Pak for applications v3.0 on top of RHOCP 3.11 cluster for Business Partners by downloading the image from Partner World. 

##### Note:
The detailed and official steps about this installation is available in knowledge center at https://www.ibm.com/support/knowledgecenter/SSCSJL_3.x/install-icpa-cli.html. But this would requires your machine to connect to ENTITLED_REGISTRY (cp.icr.io).

### Step 1:  Download following parts from PartnerWorld Software Access Catalog

https://www.ibm.com/partnerworld/program/benefits/software-access-catalog

* Click on above link
* Click on Software Access Catalog link under Overview Section.
* Look at "Find by part number" and search for "CC4GWEN"
* You will find "Container Image Archive for IBM Cloud Pak for Applications v3.0.1 Linux English (CC4GWEN )" about 1.6GB in size.
* Download the file (CIA_FOR_CPA_V3.0.1_EN_ONLY.tar.gz) locally and move to OCP Master Node at __'/cp4a/images'__ folder (as example).


----
### Step 2: Load the required images into docker registry from master node.
Run following commands from Master node to login to Openshift as well as docker registry.

If this is brand new cluster, you can give admin privilege to 'admin' user.
```
oc login  -u system:admin
oc adm policy add-cluster-role-to-user cluster-admin admin

```

Login to Openshift.

```
oc login <openshift-host-name>:8443 -u admin -p admin
docker login docker-registry.default.svc:5000  -u $(oc whoami) -p $(oc whoami -t)

```

Now load the installation images.

```
cd /cp4a/images
tar xvfz CIA_FOR_CPA_V3.0.1_EN_ONLY.tar.gz
cd ibm-cp-applications-3.0.1.1/
./import_images.sh
```
-----
Above command should load images into docker registry.

----

### Step 3: Additional Pre-req for Transformation Advisor
#### NFS Server setup.
You would need NFS server for Transformation Advisor (ta). <br/>
If you already have access to nfs server, please note down following two values  <br/>
server ip = x.x.x.x <br/>
path for ta volument = /nfsshare/ta <br/>

*Skip steps below, of you already have nfs server setup*

If you don't have existing nfs server, you can setup one quickly with following instructions.

```
yum install -y nfs-utils
mkdir /nfsshare
chmod -R 755 /nfsshare
chown nfsnobody:nfsnobody /nfsshare
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server

```

Export the share

vi /etc/exports and add following line
```
/nfsshare *(rw,sync,no_subtree_check,no_root_squash,no_all_squash)
```

Restart nfs-Server
```
systemctl restart rpcbind
systemctl restart nfs-server

```
Add share for Transformation Advisor
```
mkdir /nfsshare/ta
chmod 755 /nfsshare/ta

```


As mentioned in knowledgecenter, create a PV.
Easiest way is to create a new pv.yaml file and paste the content.
```
apiVersion: v1
kind: PersistentVolume
metadata:
 name: tanfspv
 labels:
   pvc_for_app: "ta"
spec:
 capacity:
   storage: 8Gi
 accessModes:
 - ReadWriteOnce
 nfs:
   path: /nfsshare/ta
   server: x.x.x.x
 persistentVolumeReclaimPolicy: Recycle

 ```

Run folliwng commands
```
oc create -f pv.yaml

```

### Step 4: Update the configuration for Cloud Pak installation.
```
cd /cp4a/images
mkdir data
docker run -v $PWD/data:/data:z -u 0 \
          -e LICENSE=accept \
          "cp/icpa/icpa-installer:3.0.1.1" cp -r data/* /data

```
Look at data folder, you will find four config files. <br/>
Edit config.yaml and add subdomain value.<br/>


### Install Cloud Pak for Applications.
Run this command from /cp4a/images or place where data directory is available.
```
cd ..
docker run -v ~/.kube:/root/.kube:z -u 0 \
           -t -v $PWD/data:/installer/data:z \
           -e LICENSE=accept \
           "cp/icpa/icpa-installer:3.0.1.1" install --skip-tags check-registry
```

You should see 'Installation complete' message with all URLs.

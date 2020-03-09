Installing CP4A on OCP on IBMCloud
----
These are the steps I took to install the CP4A on managed OpenShift running on IBM Cloud. 

This is very specific to the current versions and will likely be outdated information in the very near future because of changes to OCP and CP4A, however these are the steps I completed to install in OCP version 4.3.1 running on IBM Cloud.  I'm installing Cloud Pak for Apps version 4.0.1


Provision OCP on IBM Cloud.  
--
This takes between 40 mins, and 1 day depending on how well the DNS is working.  Once you have access to the console follow these steps to prep the environment before installing CP4A. 

Create project istio-system
--
You can do this from GUI or with command
`oc create project istio-system`

Login to cluster and install Service mesh (NOT THE COMMUNITY ONE)
--
Following these instructions, but summarizing and customizing for exactly what I did
https://docs.openshift.com/container-platform/4.2/service_mesh/service_mesh_install/installing-ossm.html#ossm-operator-install-istio_installing-ossm

Operators --> OperatorHub --> Search for `Service Mesh`. 
Select: *Red Hat OpenShift Service Mesh* NOT THE COMMMUNITY ONE (Should be version 1.0.8+ (sometimes the GUI shows 1.0.2, but it's actually higher)


Create Control Plane
--
Operators —> Installed Operators.   Project = istio-system.   

      Select:  Red Hat OpenShift Service Mesh —> Istio Service Mesh Control Plane —> Create ServiceMeshControlPlane (Modify YAML if you want, but I didn’t) —> Create
      
This takes about 5 minutes to complete, you can view if it's complete by running this command
`oc get smcp -n istio-system`

Create Member Roll
--
Operators —> Installed Operators.   Project = istio-system.   

      Select: Red Hat OpenShift Service Mesh —> Istio Service Mesh Member Roll —> Create ServiceMeshMemberRoll —> Modify YAML to add members, require: knative-serving, tekton-pipelines, kabanero —> Create
Here is my YAML
```YAML
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
    - knative-serving
    - tekton-pipelines
    - kabanero

```

Remove installed version of elasticsearch, it conflicts with CP4A
--
`oc delete sub elasticsearch-operator-4.3-redhat-operators-openshift-marketplace -n openshift-operators`
`oc delete clusterserviceversion elasticsearch-operator.4.3.1-202002032140 -n openshift-operators`

Download the configuration files to install CP4A
--
Important to add the `--skip-tags servicemesh` since we installed it manually. 

Follow these steps otherwise:
https://www.ibm.com/support/knowledgecenter/en/SSCSJL_3.x/install-icpa.html
```
export ENTITLED_REGISTRY=cp.icr.io
export ENTITLED_REGISTRY_USER=cp
export ENTITLED_REGISTRY_KEY=<apikey>

docker login "$ENTITLED_REGISTRY" -u "$ENTITLED_REGISTRY_USER" -p "$ENTITLED_REGISTRY_KEY"

mkdir data
docker run -v $PWD/data:/data:z -u 0 \
           -e LICENSE=accept \
           "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:4.0.1" cp -r "data/*" /data
```

Note: For **IBM employees** here are steps to get license: 
https://github.ibm.com/CloudPakOpenContent/cloudpak-entitlement
Follow same instructions as above to install, however use `ENTITLED_REGISTRY_USER: ekey` instead of `cp`


Setting the Storage class for TA
--
TA uses couchdb, which needs a PersistentVolume with RWX permissions.  There are two ways to solve this.  Easiest is to change the transadv.yml in the data directory
```
 persistence:
      accessMode: "ReadWriteOnce"
```
Alternatively you can change the storageclass to allow for ReadWriteMany. 
By default OCP on IBM Cloud uses storage class `bmc-block-bronze`, which is block storage.  For block storage you can't have RWX permission.  

To fix this, you can set the storage class to file type.  There are many options if you look in Storage --> Storage Classes. 
I chose `ibmc-file-gold-gid`.  
*Note:* ibmc-file-gold doesn't work, it doesn't have the right permissions or whatever.  Not sure exactly what the differences arem, but the -gid one works. 

You can set this in the transadv.yml in the data directory before you run the install.  Search for storageClassName: and set it to ibmc-file-gold, or whatever 
      `storageClassName: ibmc-file-gold-gid`


If you're like me and you didn't set this up during the install here is how you fix it after the fact. 
Go to:
Administration --> Custom Resource Definitions --> Search for TransAdv --> Instances --> ta --> YAML

Search for storageClassName and set it to: `ibmc-file-gold-gid`
Click Save.

If the application is already bound to a storage class you'll have to delete the old one. 
Storage --> Persistent Volumes -  Find Claim ta-XXXXXX-couchdb, click on the ... on the right and select delete
(Will not actually be removed until all pods connected are deleted/stopped)
Storage --> Persistent Volume Claims  - Find Claim ta-XXXXXX-couchdb, click on the ... on the right and select delete
(Again will not actually remove until all pods connected are deleted/stopped)
Stop the Couchdb pods:
Workloads --> Deployments  Set Project: TA, and find pod ta-XXXXXX-couchdb.  
Set the desired number of pods to 0.  This might automatically scale back up to 1, but gives OCP enough time to delete the PVC and PV, so new ones will be created with the correct configuration. 

The PV takes a couple minuted to provision, but TA should come up now. 


Installing CP4A
--
Finally!  We're ready to install CP4A 
Run the following command.  **Important** add the option `--skip-tags servicemesh` because we will use the Red Hat service mesh instead of installing our own istio. 

`docker run -v ~/.kube:/root/.kube:z -u 0 -t     -v $PWD/data:/installer/data:z     -e LICENSE=accept     -e ENTITLED_REGISTRY -e ENTITLED_REGISTRY_USER -e ENTITLED_REGISTRY_KEY     "$ENTITLED_REGISTRY/cp/icpa/icpa-installer:4.0.1" install --skip-tags servicemesh`


Easy peasy, all done!



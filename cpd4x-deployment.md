
## Topics

* 1.0 Deployment Planning
* 2.0 Setup Bastion Host
* 3.0 Registry Mirroring
* 4.0 Configuring cluster pull images
* 5.0 Node settings
* 6.0 Project creation and NamespaceScope
* 7.0 Custom Security Context Constraints
* 8.0 Creating Catalog sources
* 9.0 Creating Subscriptions
* 10.0 Installing CPD and Watson Knowledge Catalog


## 1.0 Deployment Requirements & Planning

- OpenShift 4.6.31 or later

- Cluster Compute Requirement
  - [ ] Minimum 3 compute nodes
  - [ ] Each node should be of minimum 16core, 64GB RAM and 200GB storage
  - [ ] Total Compute requirement will be calculated based on the number of services and user workload. IBM Team will help size the cluster compute requirements.
  - [ ] Minimum Compute requirement for installation: 48 Core, 196 GB
  - [ ] OpenShift nodes should meet the configuration requirements for CPD services

- Cloud Pak for Data 4.0 Services
  - [ ] IBM common Foundation Service
  - [ ] Cloud Pak for Data - Control Plane
  - [ ] Watson Knowledge Catalog and its bundle

- Networking Internet access to Mirror IBM image registries
  - [ ] docker.io/ibmcom
  - [ ] quay.io/opencloudio
  - [ ] cp.icr.io/cp 
  - [ ] icr.io/cpopen
- Networking Internet access to download IBM installer and cases:
  - [ ] https://github.com/IBM/cloud-pak-cli/releases/download
  - [ ] https://github.com/IBM/cloud-pak/raw/master/repo/case
- Storage
  - [ ] Persistent Storage 1 TB
  - [ ] Registry Storage 300 GB (example, Artifactory)
- License
  - [ ] IBM Cloud Pak for Data Standard/Enterprise license
  - [ ] Access to IBM Entitlement Key (myibm.ibm.com)
- User Permissions
  - [ ] OpenShift Administrator ( Cluster preparation )
  - [ ] Cloud Pak for Data administrator ( WKC Installation )
  - [ ] Read/Write permission to Artifactory Registry 
- Bastion Host
   - [ ] RHEL 8x, 500GB disk
   - [ ] Skopeo 1.x
   - [ ] python 2.x & 3.x
   - [ ] pyyaml, jq



## 2.0 Setup Bastion Host
Installation client machine environment setup

```
# Namespaces and Deployment configurations
export IBM_COMMON_SERVICE=ibm-common-services
export CPD_OPERATORS=ibm-common-services
export CPD_INSTANCE=cpd                     
export STORAGE_TYPE=nfs                        
export CPD_LICENSE=Enterprise                  
export STORAGE_CLASS=managed-nfs-storage       # Replace with the name of a RWX storage class
export ZEN_METADB=managed-nfs-storage          # (Recommended) Replace with the name of a RWO storage class

# Entitlement and Access tokens
export REGISTRY_USER=cp 
export REGISTRY_PASSWORD=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE1OTQ1MjAxOTgsImp0aSI6ImM4MDhjOTE3NDA3MjQ3YTJhYmFkZmFmMDI1YTQyYTM0In0.ZwWJpZqrTO8pwJP7SVcBHyWK9dFbQgH3idKTC78EyBw
export REGISTRY_SERVER=cp.icr.io

export PRIVATE_REGISTRY=ai.ibmcloudpack.com
export PRIVATE_REGISTRY_USER=admin
export PRIVATE_REGISTRY_PASSWORD=adminpass

export CASE_REPO_PATH=https://github.com/IBM/cloud-pak/raw/master/repo/case
export CASECTL_RESOLVE_DEPENDENCIES=false

export HOMEDIR=/root/cpd401
export OFFLINEDIR=${HOMEDIR}/offline

mkdir -p  ${OFFLINEDIR}
cd ${HOMEDIR}

export CASECTL_RESOLVE_DEPENDENCIES=false                         # This required for ibm-cp-datacore
export USE_SKOPEO=true
```

###  Download Installer
```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.10.0/cloudctl-linux-amd64.tar.gz
tar -xf cloudctl-linux-amd64.tar.gz
cp cloudctl-linux-amd64 /usr/bin/cloudctl
cloudctl version

# OpenShift client (only if you don't have latest oc client)
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.6.30/openshift-client-linux-4.6.30.tar.gz
tar xvf openshift-client-linux-4.6.30.tar.gz
cp oc /usr/bin
cp kubectl /usr/bin
oc version



# Skopeo 1.x version is required
sudo yum install -y httpd-tools podman ca-certificates openssl skopeo jq bind-utils git

yum install -y python2
alternatives --set python /usr/bin/python2

pip2 install pyyaml

# skopeo Version 1.2.0 or later
skopeo --version
```



##  3.0 Registry Mirroing
Mirroring images with a bastion node

### Setup private registry
private registry, This is not for production

```
podman run --privileged -d   --name registry   -p 5000:5000 -v /data/private:/var/lib/registry registry:2
```



#### Private registry access validation
```
podman login -u ${PRIVATE_REGISTRY_USER} -p ${PRIVATE_REGISTRY_PASSWORD} ${PRIVATE_REGISTRY} --tls-verify=false
```
#### IBM registry access validation
```
podman login -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD} ${REGISTRY_SERVER} --tls-verify=false
```
Following commands will save the CASE file for IBM common foundation services, Cloud Pak for Data, Common Core Services and Watson Knowledge Catalog and Mirroring to customer's private registry.

```
# cloud pak for data
cloudctl case save \
  --case ${CASE_REPO_PATH}/ibm-cp-datacore-2.0.3.tgz \
  --outputdir ${OFFLINEDIR} \
  --no-dependency

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.3.tgz \
  --inventory cpdPlatformOperator \
  --action configure-creds-airgap \
  --args "--registry cp.icr.io --user cp --pass ${REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"


cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.3.tgz \
  --inventory cpdPlatformOperator \
  --action configure-creds-airgap \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD}"

# Common services
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-cp-common-services-1.6.0.tgz \
--outputdir ${OFFLINEDIR}

# cpd scheduling
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-cpd-scheduling-1.2.2.tgz \
--outputdir ${OFFLINEDIR}

# wkc
cloudctl case save \
--case ${CASE_REPO_PATH}/ibm-wkc-4.0.1.tgz \
--outputdir ${OFFLINEDIR}

sed -i -e '/edb-postgres-advanced/d' ${OFFLINEDIR}/ibm-cloud-native-postgresql-4.0.*-images.csv
export USE_SKOPEO=true

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.3.tgz \
  --inventory cpdPlatformOperator \
  --action mirror-images \
  --args "--registry ${PRIVATE_REGISTRY} --user ${PRIVATE_REGISTRY_USER} --pass ${PRIVATE_REGISTRY_PASSWORD} --inputDir ${OFFLINEDIR}"

```
#### Validate the mirrored images
```
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} http://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool
curl -k -u ${PRIVATE_REGISTRY_USER}:${PRIVATE_REGISTRY_PASSWORD} http://${PRIVATE_REGISTRY}/v2/_catalog?n=6000 |  python -m json.tool | wc -l
```


## 4.0 Configure Global cluster pullsecret and ImageContentSourcePolicy


###  Create the utility

    
```
usage util-add-pull-secret.sh <repository> <user> <password>

# cat util-add-pull-secret.sh
#!/bin/bash

if [ "$#" -lt 3 ]; then
  echo "Usage: $0 <repo-url> <artifactory-user> <API-key>" >&2
  exit 1
fi

# set -x

REPO_URL=$1
REPO_USER=$2
REPO_API_KEY=$3

pull_secret=$(echo -n "$REPO_USER:$REPO_API_KEY" | base64 -w0)
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | sed -e 's|:{|:{"'$REPO_URL'":{"auth":"'$pull_secret'","email":"not-used"\},|' > /tmp/dockerconfig.json
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/dockerconfig.json

```

### Add pull secret for private registry
```
chmod +x util-add-pull-secret.sh
./util-add-pull-secret.sh ${PRIVATE_REGISTRY} ${PRIVATE_REGISTRY_USER} ${PRIVATE_REGISTRY_PASSWORD}

```

### Verify the pullsecrets
```
oc get secret/pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

Output (example):

 "auths": {
    "ai.ibmcloudpack.com": {
      "auth": "YWRtaW46YWRtaW5wYXNz",
      "email": "not-used"
    },
    
```

### Image content source policy
Note: This will cause the nodes to restart
Requires Cluster Admin permission
```
cat <<EOF |oc apply -f -
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: cloud-pak-for-data-mirror
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ${PRIVATE_REGISTRY}/opencloudio
    source: quay.io/opencloudio
  - mirrors:
    - ${PRIVATE_REGISTRY}/cp
    source: cp.icr.io/cp
  - mirrors:
    - ${PRIVATE_REGISTRY}/cp/cpd
    source: cp.icr.io/cp/cpd
  - mirrors:
    - ${PRIVATE_REGISTRY}/cpopen
    source: icr.io/cpopen
EOF

oc get imageContentSourcePolicy
oc get node

Wait for all the nodes are updated

oc get mcp
oc get nodes

Cluster nodes will restart, Wait for all the nodes are ready

watch "oc get nodes"

NAME                              STATUS                     ROLES    AGE   VERSION
master0.lustier.cp.fyre.ibm.com   Ready,SchedulingDisabled   master   66m   v1.19.0+d670f74
master1.lustier.cp.fyre.ibm.com   Ready                      master   66m   v1.19.0+d670f74
master2.lustier.cp.fyre.ibm.com   Ready                      master   65m   v1.19.0+d670f74
worker0.lustier.cp.fyre.ibm.com   Ready,SchedulingDisabled   worker   57m   v1.19.0+d670f74
worker1.lustier.cp.fyre.ibm.com   Ready                      worker   57m   v1.19.0+d670f74
worker2.lustier.cp.fyre.ibm.com   Ready                      worker   57m   v1.19.0+d670f74

```



## 5.0 Node Settings for Cloud Pak for Data
https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings

### CRIO settings


### Kernel parameters
Pleaser refere this documentation for Kernel parameter requirement for WKC https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=tasks-changing-required-node-settings Below settings are respective for node of 64 cpu and 791 of RAM (as per Danske setup for one node configuration, subject to change if the nodes size changes).

```
cat <<EOF |oc apply -f -
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: cp4d-wkc-ipc
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
  - name: cp4d-wkc-ipc
    data: |
      [main]
      summary=Tune IPC Kernel parameters on OpenShift Worker Nodes running WKC Pods
      [sysctl]
      kernel.shmall = 33554432
      kernel.shmmax = 68719476736
      kernel.shmmni = 32768
      kernel.sem = 250 1024000 100 32768
      kernel.msgmax = 65536
      kernel.msgmnb = 65536
      kernel.msgmni = 32768
      vm.max_map_count = 262144
  recommend:
  - match:
    - label: node-role.kubernetes.io/worker
    priority: 10
    profile: cp4d-wkc-ipc
EOF
```

### Configure Unsafe SysCtl for Db2U, custom-kubelet
```
cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: db2u-kubelet
spec:
  machineConfigPoolSelector:
    matchLabels:
      db2u-kubelet: sysctl
  kubeletConfig:
    allowedUnsafeSysctls:
      - "kernel.msg*"
      - "kernel.shm*"
      - "kernel.sem"
EOF

oc label machineconfigpool worker db2u-kubelet=sysctl
oc get machineconfigpool
```


## 6.0 Create the catalog source
```
oc get catalogsource -n openshift-marketplace

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-common-services-1.6.0.tgz \
  --inventory ibmCommonServiceOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--registry ${PRIVATE_REGISTRY} --inputDir ${OFFLINEDIR} --recursive"

oc get catalogsource -n openshift-marketplace opencloud-operators \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'


cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cpd-scheduling-1.2.2.tgz \
  --inventory schedulerSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--inputDir ${OFFLINEDIR} --recursive"

oc get catalogsource -n openshift-marketplace ibm-cpd-scheduling-catalog \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-cp-datacore-2.0.3.tgz \
  --inventory cpdPlatformOperator \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--inputDir ${OFFLINEDIR} --recursive"

oc get catalogsource -n openshift-marketplace cpd-platform \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

yum install -y python2
alternatives --set python /usr/bin/python2
pip2 install pyyaml

cloudctl case launch \
  --case ${OFFLINEDIR}/ibm-wkc-4.0.1.tgz \
  --inventory wkcOperatorSetup \
  --namespace openshift-marketplace \
  --action install-catalog \
    --args "--inputDir ${OFFLINEDIR} --recursive"


oc get catalogsource -n openshift-marketplace ibm-cpd-wkc-operator-catalog \
-o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'
```

### 7.0  Project/Namespace setup

```
oc new-project ibm-common-services
oc new-project cpd
```
### Operator Group creation
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: operatorgroup
  namespace: ibm-common-services
spec:
  targetNamespaces:
  - ibm-common-services
EOF


oc get OperatorGroup operatorgroup -n ${IBM_COMMON_SERVICE} -o yaml
```

### Namescope operators

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-namespace-scope-operator
  namespace: ${IBM_COMMON_SERVICE}
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-namespace-scope-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF

Wait for the Namescope operator pod to come up before running the next command oc get pods -n ${IBM_COMMON_SERVICE}

cat <<EOF |oc apply -f -
apiVersion: operator.ibm.com/v1
kind: NamespaceScope
metadata:
  name: cpd-operators
  namespace: ${IBM_COMMON_SERVICE}
spec:
  namespaceMembers:
  - ${IBM_COMMON_SERVICE}
  - ${CPD_INSTANCE}
EOF



oc -n ibm-common-services get namespacescope common-service -o yaml
oc -n ibm-common-services edit namespacescope common-service
oc -n ibm-common-services get configmap namespace-scope -o yaml

```

### ConfigMap for Deploying in custom namespaces

IBM Common services are deployed in the namespace "ibm-common-services", Cloud Pak for Data operators are in "cpd-operators" by default. ConfigMap is required when customer wants to use custom namespaces.
```
cat <<EOF |oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: common-service-maps
  namespace: kube-public
data: 
  common-service-maps.yaml: |
    namespaceMapping:
    - requested-from-namespace:
      - ${CPD_INSTANCE}
      map-to-common-service-namespace: ${IBM_COMMON_SERVICE} 
    defaultCsNs: ibm-common-services
EOF

```


## 8.0 Creating Subscriptions


### IBM Cloud Pak foundational services
v3.10.0
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-common-service-operator
  namespace: ibm-common-services
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-common-service-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF

oc --namespace ibm-common-services get csv
oc get crd | grep operandrequest
oc api-resources --api-group operator.ibm.com
```

### IBM Cloud pack for Data Scheduler (Optional)
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-cpd-scheduling-catalog-subscription
  namespace: ibm-common-services   # Specify the project that contains the Cloud Pak foundational services operators
spec:
  channel: v1.2
  installPlanApproval: Automatic
  name: ibm-cpd-scheduling-operator
  source: ibm-cpd-scheduling-catalog
  sourceNamespace: openshift-marketplace
EOF

oc get sub -n cpd-operators ibm-cpd-scheduling-catalog-subscription \
-o jsonpath='{.status.installedCSV} {"\n"}'

oc get csv -n cpd-operators ibm-cpd-scheduling-operator.v1.2.2 \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'

oc get deployments -n cpd-operators -l olm.owner="ibm-cpd-scheduling-operator.v1.2.2" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```

### Cloud Pak for Data operator subscription

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cpd-operator
  namespace: ibm-common-services   #  Pick the project where you want to install the Cloud Pak for Data platform operator
spec:
  channel: v2.0
  installPlanApproval: Automatic
  name: cpd-platform-operator
  source: cpd-platform
  sourceNamespace: openshift-marketplace
EOF
```

### NamespaceScope Operator
```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-namespace-scope-operator
  namespace: ibm-common-services
spec:
  channel: v3
  installPlanApproval: Automatic
  name: ibm-namespace-scope-operator
  source: opencloud-operators
  sourceNamespace: openshift-marketplace
EOF


oc get sub -n ibm-common-services cpd-operator \
-o jsonpath='{.status.installedCSV} {"\n"}'

oc get csv -n ibm-common-services cpd-platform-operator.v2.0.3 \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'


oc get deployments -n ibm-common-services -l olm.owner="cpd-platform-operator.v2.0.3" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```


### WKC Subscription creation

```
cat <<EOF |oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    app.kubernetes.io/instance:  ibm-cpd-wkc-operator-catalog-subscription
    app.kubernetes.io/managed-by: ibm-cpd-wkc-operator
    app.kubernetes.io/name:  ibm-cpd-wkc-operator-catalog-subscription
  name: ibm-cpd-wkc-operator-catalog-subscription
  namespace: ibm-common-services   # Pick the project that contains the Cloud Pak for Data operator
spec:
    channel: v1.0
    installPlanApproval: Automatic
    name: ibm-cpd-wkc
    source: ibm-cpd-wkc-operator-catalog
    sourceNamespace: openshift-marketplace
EOF


oc get sub -n ibm-common-services  ibm-cpd-wkc-operator-catalog-subscription \
-o jsonpath='{.status.installedCSV} {"\n"}'

oc get csv -n ibm-common-services ibm-cpd-wkc.v1.0.1 \
-o jsonpath='{ .status.phase } : { .status.message} {"\n"}'


oc get deployments -n ibm-common-services -l olm.owner="ibm-cpd-wkc.v1.0.1" \
-o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
```


## 9.0 Installation

### Installing individual foundational services
```
The IBM Cloud Pak for Data platform operator automatically installs the following foundational services:
* Certificate management service
* Identity and Access Management Service (IAM Service)
* Administration hub
```


### NamespaceScope Operator

```
cat <<EOF |oc apply -f -
apiVersion: operator.ibm.com/v1
kind: NamespaceScope
metadata:
  name: cpd-operators
  namespace: ibm-common-services       # (Default) Replace with the Cloud Pak for Data platform operator project name
spec:
  csvInjector:                    # This setting is required for some services. Do not delete this line if you specified it when you created operator subscriptions.
    enable: true                  # This setting is required for some services. Do not delete this line if you specified it when you created operator subscriptions.
  namespaceMembers:
  - ibm-common-services                # (Default) Replace with the Cloud Pak for Data platform operator project name
  - cpd                           # Replace with the project where you will install Cloud Pak for Data
EOF
```

### Installing Cloud Pak for Data
```
cat <<EOF |oc apply -f -
apiVersion: cpd.ibm.com/v1
kind: Ibmcpd
metadata:
  name: ibmcpd-cr                                     # This is the recommended name, but you can change it
  namespace: cpd                                      # Replace with the project where you will install Cloud Pak for Data
spec:
  license:
    accept: true
    license: Enterprise                               # Specify the Cloud Pak for Data license you purchased
  storageClass: managed-nfs-storage                   # Replace with the name of a RWX storage class, such as managed-nfs-storage
EOF



oc get Ibmcpd ibmcpd-cr -o jsonpath="{.status.controlPlaneStatus}{'\n'}" -n cpd
oc get ZenService lite-cr -o jsonpath="{.status.zenStatus}{'\n'}" -n cpd

# Wait for the ZenService status to become completed

# On Install completion, you can get the CPD URL with the following command
oc get ZenService lite-cr -o jsonpath="{.status.url}{'\n'}" -n cpd

# Get password
oc extract secret/admin-user-details --keys=initial_admin_password --to=- -n cpd
```

### Installing Watson Knowledge Catalog

```
cat <<EOF |oc apply -f -
apiVersion: wkc.cpd.ibm.com/v1beta1
kind: WKC
metadata:
  name: wkc-cr     # This is the recommended name, but you can change it
  namespace: cpd    # Replace with the project where you will install Watson Knowledge Catalog
spec:
  license:
    accept: true
    license: Enterprise                      # Specify the license you purchased
  version: 4.0.1
  storageClass: managed-nfs-storage          #if you use a different storage class, replace it with the appropriate storage class
  # wkc_db2u_set_kernel_params: True
  # iis_db2u_set_kernel_params: True
  # install_wkc_core_only: true     # To install the core version of the service, remove the comment tagging from the beginning of the line.

EOF

oc get WKC wkc-cr -o jsonpath='{.status.wkcStatus} {"\n"}'
oc get CCS ccs-cr -o jsonpath='{.status.ccsStatus} {"\n"}'
oc get DataRefinery datarefinery-sample -o jsonpath='{.status.datarefineryStatus} {"\n"}'
oc get Db2aaserviceService db2aaservice-cr -o jsonpath='{.status.db2aaserviceStatus} {"\n"}'
oc get IIS iis-cr -o jsonpath='{.status.iisStatus} {"\n"}'
oc get UG ug-cr -o jsonpath='{.status.ugStatus} {"\n"}'
```

### Resize PVC
```
# Change the storage value in "spec" from 40Gi to 100Gi
oc edit pvc data-c-db2oltp-iis-db2u-0
```

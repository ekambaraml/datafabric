##  Deployment planning
Requirements

    OpenShift 4.6.31 or later

    Cluster Compute Requirement
        Minimum 3 compute nodes
        Each node should be of minimum 16core, 64GB RAM and 200GB storage
        Total Compute requirement will be calculated based on the number of services and user workload. IBM Team will help size the cluster compute requirements.
        Minimum Compute requirement for installation: 48 Core, 196 GB
        OpenShift nodes should meet the configuration requirements for CPD services

    Cloud Pak for Data 4.0 Services
        Cloud Pak for Data - Control Plane
        DB2U Database
        Watson Knowledge Catalog and its bundle

    Networking Internet access to Mirror IBM image registries
        docker.io/ibmcom
        quay.io/opencloudio
        cp.icr.io/cp
        icr.io/cpopen

    Networking Internet access to download IBM installer and cases:
        https://github.com/IBM/cloud-pak-cli/releases/download
        https://github.com/IBM/cloud-pak/raw/master/repo/case

    Storage
        Persistent Storage 1 TB
        Registry Storage 300 GB (example, Artifactory)

    License
        IBM Cloud Pak for Data Standard/Enterprise license
        Access to IBM Entitlement Key (myibm.ibm.com)

    User Permissions
        OpenShift Administrator ( Cluster preparation )
        Cloud Pak for Data administrator ( WKC Installation )
        Read/Write permission to Artifactory Registry

    Bastion Host
        RHEL 8x, 500GB disk
        Skopeo 1.x
        python 2.x & 3.x
        pyyaml, jq



## Setup Install client environment

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

## Download Installer
```
wget https://github.com/IBM/cloud-pak-cli/releases/download/v3.10.0/cloudctl-linux-amd64.tar.gz
tar -xf cloudctl-linux-amd64.tar.gz
cp cloudctl-linux-amd64 /usr/bin/cloudctl
cloudctl version

# Skopeo 1.x version is required
sudo yum install -y httpd-tools podman ca-certificates openssl skopeo jq bind-utils git

yum install -y python2
alternatives --set python /usr/bin/python2

pip2 install pyyaml

# skopeo Version 1.2.0 or later
skopeo --version
```

## Setup private registry
podman run --privileged -d   --name registry   -p 5000:5000 -v /data/private:/var/lib/registry registry:2



## Mirroring images with a bastion node

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



## OpenShift Configuration



## Node Settings
```
# Kernel parameters
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


# Kubectl
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




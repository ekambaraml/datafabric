
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
```

## 

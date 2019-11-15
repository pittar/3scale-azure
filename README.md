# Installing 3Scale 2.6 on Azure

These instructions are meant to quickly deploy 3Scale 2.6 on OpenShift 3.11 on Azure, using Azure File as the RWX Storage Class.

This has **not** been tested for OpenShift 4.x and there are currently permission issues with Azure File.  

**Use at your own risk!**

## Project and Registry Access Token

First, create a new project:
```oc new-project 3scale```

Next, you will need a registry access token to pull images from the Red Hat registry.
* Create a [service account here](https://access.redhat.com/terms-based-registry) and download the `OpenShift Secret` to a local directory.

Create this secret in your 3Scale project:
```oc apply -f <token yaml file name> -n 3scale```

You now have a `3Scale` project with a pull token for the Red Hat container registry.

## Azure File (RWX) StorageClass

3Scale requires one `RWX` volume.  By default, OpenShift doesn't have RWX volumes configured.  Instead of setting up OpenShift Container Storage or an NFS server, you can user Azure File.

You need to create a new `Role`, `RoleBinding` and `StorageClass`, as well as add the `admin` user to the role.

First, create file named `azure-file-role.yaml` containing:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: system:controller:persistent-volume-binder
  namespace: 3scale
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "delete"]
```

Apply this role:
```
oc apply -f azure-file-role.yaml -n 3scale
```

Create the role binding file `azure-file rolebinding.yaml`:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: system:controller:persistent-volume-binder
  namespace: 3scale
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: system:controller:persistent-volume-binder
subjects:
- kind: ServiceAccount
  name: persistent-volume-binder
namespace: kube-system
```

Apply this RoleBinding:
```
oc apply -f azure-file-rolebinding.yaml -n 3scale
oc policy add-role-to-user admin system:serviceaccount:kube-system:persistent-volume-binder -n 3scale
```

Finally, create the new `StorageClass` by creating and applying a file named `azure-file-storageclass.yaml`:
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
```

```
oc apply -f azure-file-storageclass.yaml -n 3scale
```

## Create the 3Scale Eval Template

The 2.6 Eval template is a full 3Scale installation, but with lower resource requirments.  The assumption being an eval installation won't be recieving production-level traffic.

```
oc create -f https://raw.githubusercontent.com/3scale/3scale-amp-openshift-templates/2.6.0.GA/amp/amp-eval-tech-preview.yml -n 3scale
```

Next, instantiate the template to start the 3Scale install process.

```
oc new-app 3scale-api-management-eval \
    -p TENANT_NAME=azure \
    -p WILDCARD_DOMAIN=40.121.63.74.nip.io \
    -p ADMIN_PASSWORD=password \
    -p MASTER_PASSWORD=password \
    -p RWX_STORAGE_CLASS=azuzrefile
```




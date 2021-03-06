# Managing Configurations

## How to update configurations

Kubernetes resources for WSO2 products use Kubernetes [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
to pass on the minimum set of configurations required to setup a product deployment pattern.

For example, the minimum set of configurations required to setup pattern-distributed of WSO2 Stream Processor can be found in `<KUBERNETES_HOME>/pattern-distributed/confs`
directory. The Kubernetes ConfigMaps are generated from these files.

If you intend to pass on any additional configuration changes, you may use Kubernetes ConfigMaps. Follow the 
steps below to achieve it.

**[1] In order to apply the updated configurations, WSO2 product server instances need to be restarted. Hence, un-deploy all the Kubernetes resources
corresponding to the product deployment, if they are already deployed.**

**[2] Create a Kubernetes ConfigMap from the file(s), which contains the relevant configuration changes.**

The need to create a Kubernetes ConfigMap may depend on the type of file(s) to be passed on to the cluster, as follows:

***[i] If the additional configuration is part of a file, which is among the minimum set of files with configuration changes required to setup
the particular product deployment pattern, use the same copy of the file to pass on the configuration.***

e.g. `<KUBERNETES_HOME>/pattern-distributed/confs/sp-manager/conf/deployment.yaml` is a file which is part of the minimum set of files with configuration changes required for
pattern-distributed of WSO2 Stream Processor. If you intend to make the configuration change in the `<WSO2_SP_HOME>/conf/manager/deployment.yaml`
file in the product pack (which is the original file corresponding to `<KUBERNETES_HOME>/pattern-distributed/confs/sp-manager/conf/deployment.yaml` file),
make the configuration change within the file copy `<KUBERNETES_HOME>/pattern-distributed/confs/sp-manager/conf/deployment.yaml`.

***[ii] If the additional configuration file is not included among the minimum set of files with configuration changes required to setup
a particular product deployment pattern, but is part of a directory within the original product pack to which you already pass other configuration files
using a Kubernetes ConfigMap, include the file within the appropriate location in `<KUBERNETES_HOME>/pattern-distributed/confs` folder or any of its sub-folders.***

e.g. Assume that you need to change a configuration in `<WSO2_SP_HOME>/conf/manager/log4j2.xml` file.
`<WSO2_SP_HOME>/conf/manager/log4j2.xml` is not among the minimum set of configuration files adjusted
for pattern-distributed of WSO2 Stream Processor. A Kubernetes ConfigMap is already created from `<KUBERNETES_HOME>/pattern-distributed/confs/sp-manager/conf` folder,
passing configuration files to `<WSO2_SP_HOME>/conf/manager/` in the original product pack. Hence, you can add a copy of the `log4j2.xml`
with relevant changes to `<KUBERNETES_HOME>/pattern-distributed/confs/sp-manager/conf` folder, in order to pass on the configuration file.

***[iii] If the additional configuration file is not included among the minimum set of files with configuration changes required to setup a particular product
deployment pattern and is **not** part of any directory within the original product pack to which you already pass other configuration files
using Kubernetes ConfigMaps, follow the steps given below along with appropriate examples in each step.***

For example, assume that you need to pass on a copy of the changed `<WSO2_SP_HOME>/conf/osgi/launch.properties` file
to the Kubernetes cluster, for pattern-distributed of WSO2 Stream Processor. `<PATH_TO_CONFIG_FILE>` is the path to a local copy of
`<WSO2_SP_HOME>/conf/osgi/launch.properties` file.

* Create a folder in your local machine's filesystem, add the file with configuration changes to the created folder and
create a Kubernetes ConfigMap.

e.g.

```
# create a folder
mkdir config

# copy the changed configuration file to the created folder
cp <PATH_TO_CONFIG_FILE> config

# create a Kubernetes ConfigMap
kubectl create configmap sp-conf-osgi --from-file=config/
```

* Populate a volume with data stored in the created Kubernetes ConfigMap. For this purpose, update the appropriate
Kubernetes [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) resource(s). When mounting
the created Kubernetes ConfigMap at the product container, it has to be mounted to the relevant path within the
`/home/wso2carbon/wso2-config-volume` folder in the product container, while maintaining the appropriate WSO2 product home folder structure.

e.g. Update the volumes' (`spec.template.spec.volumes`) and volume mounts' (`spec.template.spec.containers[wso2sp-manager].volumeMounts`) sections in
`<KUBERNETES_HOME>/pattern-distributed/sp/wso2sp-manager-1-deployment.yaml` file. The `mountPath` (which is `/home/wso2carbon/wso2-config-volume/conf/manager/osgi`)
has been derived based on the target folder structure within the original product pack (which is `<WSO2_SP_HOME>/conf/manager/osgi`) and assuming that
`/home/wso2carbon/wso2-config-volume` is the product home root folder.

```
volumeMounts:
...
- name: sp-conf-osgi-volume
  mountPath: "/home/wso2carbon/wso2-config-volume/conf/manager/osgi"

volumes:
...
- name: sp-conf-osgi-volume
  configMap:
    name: sp-conf-osgi
```

**[3] Deploy the Kubernetes resources as defined in section **Quick Start Guide** for the relevant deployment pattern.**

## How to introduce additional artifacts

If you intend to pass on any additional artifacts such as, third-party libraries, OSGi bundles and security related artifacts to the Kubernetes cluster,
you may mount the desired content to `/home/wso2carbon/wso2-artifact-volume` directory path within a WSO2 product Docker container.

The following example depicts how this can be achieved when passing additional artifacts to WSO2 Stream processor nodes
in a clustered deployment of WSO2 Stream processor:

**[1] In order to apply the updated configurations, WSO2 product server instances need to be restarted. Hence, un-deploy all the Kubernetes resources
corresponding to the product deployment, if they are already deployed.**

**[2] Create and export a directory within the NFS server instance.**
   
**[3] Add the additional third-party libraries, OSGi bundles and security related artifacts, into appropriate
folders matching that of the relevant WSO2 product home folder structure, within the previously created directory.**

**[4] Grant ownership to `wso2carbon` user and `wso2` group, for the directory created in step [2].**
      
   ```
   sudo chown -R wso2carbon:wso2 <directory_name>
   ```
      
**[5] Grant read-write-execute permissions to the `wso2carbon` user, for the directory created in step [2].**
      
   ```
   chmod -R 700 <directory_name>
   ```

**[6] Map the directory created in step [2] to a Kubernetes [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
in the persistent volume resource file `<KUBERNETES_HOME>/pattern-distributed/volumes/persistent-volumes.yaml`**

For example, append the following entry to the file:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sp-additional-artifact-pv
  labels:
    purpose: sp-additional-artifacts
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: <NFS_SERVER_IP>
    path: "<NFS_LOCATION_PATH>"
```

Provide the appropriate `NFS_SERVER_IP` and `NFS_LOCATION_PATH`.

**[7] Create a Kubernetes Persistent Volume Claim to bind with the Kubernetes Persistent Volume defined in step [6].**

For example, append the following entry to the file `<KUBERNETES_HOME>/pattern-distributed/sp/wso2sp-mgt-volume-claim.yaml`:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sp-additional-artifact-volume-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  selector:
    matchLabels:
      purpose: sp-additional-artifacts
```

**[8] Update the appropriate Kubernetes [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) resource(s).**

For example in the discussed scenario, update the volumes (`spec.template.spec.volumes`) and volume mounts (`spec.template.spec.containers[wso2sp-manager].volumeMounts`) in
`<KUBERNETES_HOME>pattern-distributed/sp/wso2sp-manager-1-deployment.yaml` file as follows:

```
volumeMounts:
...
- name: sp-additional-artifact-storage-volume
  mountPath: "/home/wso2carbon/wso2-artifact-volume"

volumes:
...
- name: sp-additional-artifact-storage-volume
  persistentVolumeClaim:
    claimName: sp-additional-artifact-volume-claim
```

**[9] Deploy the Kubernetes resources as defined in section **Quick Start Guide** for pattern-distributed of WSO2 Stream Processor.**
# Backup and Restore of Applications running on RedHat OpenShift using Velero

In this document, we will describe how to back up and restore your containerized applications running on RedHat OpenShift (RHOS) environment using Velero.

## Acknowledgements
This couldn't have been possible without the relentless work and collaboration across IBM business units. Specifically; acknowledging the contributions of [Sandeep Mangalath](https://w3.ibm.com/bluepages/profile.html?uid=05714N744) who has provided guidance specifically on use of Operators to automate backup and recovery operations using Velero. Thank you all for your unwavering support.

### Introduction to Veloro

[Veloro](https://github.com/vmware-tanzu/velero) is an open source, kubernetes backup recovery utility laregely contributed by Heptio team. As of writing this article, Heptio has been acquired by VMware and the community contributors have aggressively worked on the assimilation of Velero as a first class citizen in VMware's Tanzu mission critical portfolio.

Velero provides backup and restore capabilities for all or part of your kubernetes cluster. It backs up all tags, deployments, persistent volumes, and more. Since v0.6.0, Velero has adopted a plugin model which enables anyone to easily implement additional object and block storage backends, outside of the main Velero repository. This allows 

Velero natively supports backing up and restoring data from any Kubernetes volume or persistent volume and.can be used to restore individual crds, or deamon set; practically any kubernetes resources via Custom Resource Definitions.  Each Velero operation -- on-demand backup, scheduled backup, restore -- is a custom resource, defined with a Kubernetes Custom Resource Definition (CRD) and stored in etcd. Velero also includes controllers that process the custom resources to perform backups, restores, and all related operations.

This facilitates back up or restore of all objects in the cluster, or more granular filtering of objects by type, namespace, and/or label.

Velero is ideal for the disaster recovery use case, as well as for snapshotting your application state, prior to performing system operations on your cluster (e.g. upgrades).


* Supports multiple storage backends, including IBM Cloud Object Storage, Amazon S3, Google Cloud Storage, Azure Blob Storage, and Minio amongst others
* Fully encrypts backup data at rest and in transit with AES-256 in counter mode
* Only backs up data that has changed since the prior backup, using content-defined chunking
* De-duplicates data within a single backup for efficient use of storage


![](/ct_quote.png)


### Scope

This guide provides an overview and guidance on when it is applicable to use Velero for backup and restore operations. It also provides an IBM opinated view on the Velero product, its key capabilities and future outlook. 

It does not provide yet another recipe of "How to install and run Velero in a kubernetes enviornment" be it on 
1. "OpenShift environment"  [referenced here](https://medium.com/ibm-garage/how-to-install-velero-in-an-openshift-environment-f7484fabbbe4)  or
2. "IBM Cloud Kubernetes Service  (IKS) environment" [referenced here](https://medium.com/@mlrborowski/using-ark-and-restic-to-provide-dr-for-ibm-kubernetes-service-cae53cfe532)
2. "IBM Cloud Private environment" [referenced here](https://github.com/jchawla-git/icp-backup/blob/master/docs/backup_restore_using_ark_and_restic.md)

These previously published articles confirm that:
1. The underlying framework including kubernetes is exactly the same in OCP and IKS as the open sourced kubernetes project; supporting portability across public, private and multi-cloud environments.
2. Velero is storage agnostic.

Furthure, in demonstrating how it works; this guide derives the learnings from using Velero to backup and restore a realistic stateful application that is more commonly encountered in an enterprise setting.

With the adoption of the Operator pattern this guide oulines the use of Operators to automate Day 2 operational tasks such as back up through the use of Operators.


To briefly introduce, Operators enables a fundamentally new way to automate infrastructure and application management tasks using Kubernetes as the automation engine. With Operators, developers and Kubernetes administrators can gain the automation advantages of public cloud-like services, including provisioning, scaling, and backup/restore, while enabling the portability of the services across Kubernetes environments regardless of the underlying infrastructure.


We will set up and configure the Velero client on a local machine, and deploy the Velero server into IKS Kubernetes cluster. We'll then deploy a sample Wordpress application that uses a Persistent Volume for hosting the MySQL database, backup the application to IBM Cloud Object Storage, simulate a disaster recovery scenario and restore the application with its persistent volume.

Next we will Setup a CouchDB application using Kubernetes Operators leveraging Custom Resource Definitions (CRD), install Operator Lifecycle Manager (OLM) for Operators as a pre-requisite and use Velero to backup the CouchDB cluster.

********************

## Usage Guidance

This article does not attempt to compare Velero with other traditional backup and recovery tools; but rather guides the reader on its applicability in next gen architectures based on containers and kubernetes.

While GitOps as a methodology triggering operational tasts including backup and restore are gaining popularity with Site Reliability Engineering (SRE) team; Velero caters to the Traditional IT Operations team in providing snapshot based full and incremental backups in a Cloud native container architecture built using native Kubernetes APIs.

Furthur one important thing to note is that GitOps can only restore Kubernetes objects so that means any persistent data required for an application to correctly function must be restored for stateful applications, such as databases, to be back in service.

###Multi-architecture Container Image Support
Velero also provides [multi-arch container images support](https://velero.io/blog/velero-1.3-voyage-continues/) by using Docker manifest lists which adds Linux on Power systems support.

###Physical vs. Logical Backups

Apart from Physical backups; logical backups are an important topic to address here as it save time on operational backup tasks, especially in the case of databases to provide a point-in-time application consistent backup. These are triggered on a per workload basis and only extracts the data from the data files into dump files.

Velero supports application consistent backups via the construct of [hooks](https://velero.io/docs/v1.3.2/hooks/). When performing a backup, a user can specify one or more commands to execute in a container in a pod when that pod is being backed up. The commands can be configured to run before any custom action processing ("pre" hooks), or after all custom actions have been completed and any additional items specified by custom action have been backed up ("post" hooks).

Thus the guidance to use Velero when traditional IT teams prefer to back up their containerized applications orchestrated by kubernetes using cloud native  backup and recovery tools based on next gen architectures.


## Solution Overview

Our current setup includes three worker nodes on IKS, and one storage (NFS) node.

In order to follow all of the recommendations in this guide, it is assumed that you have already provisioned an IKS cluster and set up NFS storage for the same, and are able to have access to your cluster immediately post-install.

![Velero Backup Solution Overview](/ark_flow.png)
A simple overview of the process is as follows:

* Login (or first create) to your IBM Cloud Account.
* Create and configure IBM object storage service.
* Install Ark Client.
* Configure Ark and Restic.

* Login to your ICP cluster
* Install Ark and Restic into your ICP cluster.
* Deploy an application and make a change to the PV content.
* Run Ark backup.
* Delete the application and PV, simulating disaster.
* Restore application from Ark/Restic Backup and all is well again.

## Task 1: Setup your Backup target

We will use the IBM Cloud Object Storage (COS) service as the backup target.

## Step 1. Login to the IBM Cloud (or create you free account if this is your first time)

https://console.cloud.ibm.com

## Step 2. Create an IBM Cloud Object Storage Service Instance

To store Kubernetes backups, you need a destination bucket in an instance of Cloud Object Storage (COS) and you have to configure service credentials to access this instance.

If you don’t have a COS instance, you can create a new one, according to the detailed instructions in Creating a new resource instance. The next step is to create a bucket for your backups. Ark and Restic will use the same bucket to store K8S configuration data as well as Volume backups. See instructions in Create a bucket to store your data. We are naming the bucket arkbucket and will use this name later to configure Ark backup location. You will need to choose another name for your bucket as IBM COS bucket names are globally unique. Choose “Cross Region” Resiliency so it is easy to restore anywhere.

![COS Bucket Creation (arkbucket shown but create restic bucket also)](/icos_create_bucket.png)


The last step in the COS configuration is to define a service that can store data in the bucket. The process of creating service credentials is described in Service credentials. Several comments:

```
Your Ark service will write its backup into the bucket, so it requires the “Writer” access role.
Ark uses an AWS S3 compatible API. Which means it authenticates using a signature created from a pair of access and secret keys — a set of HMAC credentials. You can create these HMAC credentials by specifying {“HMAC”:true} as an optional inline parameter. See step 3 in the Service credentials guide.
```

![COS Service Credentials](/icos_service_credentials.png)

After successfully creating a Service credential, you can view the JSON definition of the credential. Under the ```cos_hmac_keys``` entry there are ```access_key_id``` and ```secret_access_key```. We will use them later.



## Task 2: Setup Velero


## Step 3. Download and Install the Velero Client

Download Velero as described [here](https://github.com/vmware-tanzu/velero/releases/tag/v1.3.2). A single tar ball download should install the Velero client program along with the required configuration files for your cluster.
Extract the tar.gz file and move it to a local working directory. Add Velero directory to PATH. For example if you have downloaded velero-v1.3.0-darwin-amd64 then run the below command.

```export PATH=$PATH:/usr/local/bin/velero-v1.3.0-darwin-amd64/
```

## Step 4. Configure Velero Setup

Configure your kubectl client to access your IKS deployment. Create a Velero specific credentials file (credentials-velero) in your local directory. Replace your <COS access key ID> and <COS secret Access key> with the COS service credential that you created for your COS bucket.

```
echo "[default] 
aws_access_key_id = <COS access key ID> 
aws_secret_access_key = <COS secret Access key>" > credentials-velero
```
The file should look something like this when done.

Now install install Velero server using the Velero install command as below. Replace the <COS bucket name> with your COS bucket name. Edit the COS region and s3URL to match your choices. Plugin is required as shown below.

```
velero install \
    --provider aws \
    --bucket <COS bucket name>  \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=us- east,s3ForcePathStyle="true",s3Url=https://<S3 URL>
    --plugins velero/velero-plugin-for-aws:v1.0.0
```
This will execute a Velero install on the IKS cluster as a Deployment as shown below. The output should look something like the image below when done.

![Velero install](/Velero_Install.png)

The Velero deployment creates a separate namespace called Velero and the Velero pod resides within that namespace.

![Velero PoD](/Velero_Pod.png)

## Task 3: Setup your Application for Backup

## Step 5. Deploy a sample Application with a Volume to be Backed Up

Create a PVC to associate with Storage class ibmc-file-silver


```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: mysql-silver-pvc
 labels:
   billingType: hourly
   region: us-south
   zone: dal10
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 24Gi
 storageClassName: ibmc-file-silver
 ```

The PVC gets bounded with the storage class ibmc-file-silver

![MySQL PVC](/MySQL_PVC.png)
 
We will deploy wordpress with MySQL and below is the yaml. Create a .yaml file with your choice of editor. Replace the env value for <password> with your choice of a secure password. This password will be used to login to the MySQL DB later. Run .yaml file with kubectl command.

 ```
$ kubectl apply -f <mysql.yaml>
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: <password>
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: pvc-3cb5302d-5cb5-43ed-bff3-4926f1d4852d
          mountPath: /var/lib/mysql
      volumes:
      - name: pvc-3cb5302d-5cb5-43ed-bff3-4926f1d4852d
        persistentVolumeClaim:
          claimName: mysql-silver-pvc
 ```
 
The application pod get spawned and would look similar to the image below.
 
 ![MySQL Pod](/MySQL_Pod.png)
 
Now that we have a stateful application pod created, we will examine the pod if its running and verify the details of the pods. The detailed pod description will show all information like the node, PVC, IP details, Volume mounts etc. as below.


![MySql POD describe](/MySql_POD_describe.png)


***********Sandeep*********
## Step 6. Configure the MySQL POD

We can backup up our sample application by scoping the backup to the application’s namespace. As we have deployed our application in the default namespace we can leave that as default. 

We will exec into the shell of the pod. Login to the MySQL database as a user ‘root’ and the <password> which we provided in the deployment yaml file. We will then create  few example Databases as shown below (test1, test2) which creates a persistent data in the Volume. 
 
 ![MySQL Pod Exec DB creation](/MySQL_Pod_Exec_DB_creation.png)
 
 ## Step 7. Use Velero to backup K8S config and volume
 
Use Velero command to create backup. Velero provides a granular way to find pods in different namespaces. Here we can use the Pod Selector app=wordpress which is already declared during the MySQL Deployment. This will let Velero create backup a specific pod with the matching Selector. The image below also provides the detailed description of the backup which is created. 
 
  ![MySQL Velero Backup creation](/MySQL_Velero_backup_creation.png)
  
  ## Step 8. Verify backup taken in COS
  
The below image shows the files that’s been backed up for the MySQL pod. Look closely to understand the files which are been backed up by Velero

  ![MySQL COS Backup](/MySQL_COS_backup_files.png)

 
 ## Task 4: Simulate Disaster and Restore Application
 
 ## Step 9. Simulating Disaster
We will now delete the MySQL deployments and verify if the deployments and the associated pods are also deleted as shown below.
 
 ![MySQL delete](/MySQL_delete.png)

## Step 10. Restoring using Velero 
We will run Velero restore command to restore the MySql deployment from the backup file in COS. Within few moments we notice that Velero has restored the deployment successfully. 

 ![MySQL Velero Restore DB check](/MySQL_Velero_Restore_DB_check.png)
 
 ## Step 11. Verifying the Deployment and MySQL DB 
Once we have the deployment restored and the associated Pods have come up. We exec into the shell and login into the MySQL database. We can see that the Databases we created earlier(test1 and test2) are intact. 

## Task 5: Setup an Application using K8s Operators leveraging CRD

Kubernetes has controllers which are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state. 
Kubernetes also has custom controllers called CRD which defines custom resources and the associated controllers. On their own, custom resources simply let you store and retrieve structured data. When you combine a custom resource with a custom controller, custom resources provide a true declarative API. A declarative API allows you to declare or specify the desired state of your resource and tries to keep the current state of Kubernetes objects in sync with the desired state. The controller interprets the structured data as a record of the user’s desired state, and continually maintains this state. 

An Operator is a method of packaging, deploying and managing a Kubernetes application

Thus, while a custom controller acts and listens on native Kubernetes resources and events, an Operator works with Custom Resource Definitions (CRDs), resources that the user creates to solve complex business problems. An Operator can be thought of as a replacement to the human operator. Someone who is aware of the technical details of the application and also knows the business requirements and acts accordingly.

## Step 12. Installation of OLM and CouchDB Operator 

We will deploy an Operator for CouchDB installation on IKS. CouchDB installation requires Operators and few other pre-requisites. 
The CouchDB Operator manages every step of installation of the CouchDB cluster. Thereafter it manages the health and cluster quorum of the CouchDB Cluster. 

To install Operators you must install Operator Lifecycle Manager (OLM), a tool from Red Hat to help manage the Operators running on your cluster. Run following commands to install OLM and CouchDB Operator

 ```
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/$OLM_RELEASE/install.sh | bash -s $OLM_RELEASE

kubectl create -f https://operatorhub.io/install/couchdb-operator.yaml

kubectl get csv -n operators
 ```
The Operator for Couch DB is installed on a separate namespaces as shown below in the image(see the highlighted row in the image). Also see the CouchDB Operator pod which is deployed within the Operators namespace.

 ![CouchDB Operator](/CouchDB_Operator.png)
 
## Step 13. Installation of CouchDB cluster

Using the CouchDB Operator we install CouchDB cluster. The CouchDB cluster is deployed on a separate namespace. The details of the CouchDB cluster and associated pods of the cluster is shown below.

  ![CouchDB on IKS](/CouchDB_on_IKS.png)

## Step 14. Velero backup of CouchDB cluster

We run the same Velero command to backup the CouchDB cluster. Notice that we are using the --include-namespace option to specify the cluster. We don’t need to use a pod selector as all pods are part of the similar CouchDB cluster. 

 ![CouchDB Velero backup creation](/CouchDB_Velero_backup_creation.png)

Below is the image from the COS bucket which shows CouchDB backed up files. 
 
 ![CouchDB COS backupfiles](/CouchDB_COS_backupfiles.png)

This proves the use of Operators to backup  a CouchDB cluster using Velero

## Limitations

Presently known limitations include:

1. Velero only supports a single set of credentials per provider. It's not yet possible to use different credentials for different locations, if they're for the same provider.

2. Volume snapshots are still limited by where your provider allows you to create snapshots. For example, AWS and Azure do not allow a user to create a volume snapshot in a different region than where the volume is. If a user tries to take a Velero backup using a volume snapshot location with a different region than where the cluster's volumes are, the backup will fail.

3. Each Velero backup has one BackupStorageLocation, and one VolumeSnapshotLocation per volume provider. It is not possible (yet) to send a single Velero backup to multiple backup storage locations simultaneously, or a single volume snapshot to multiple locations simultaneously. However, a user can always set up multiple scheduled backups that differ only in the storage locations used if redundancy of backups across locations is important.

4. Cross-provider snapshots are not supported. For a cluster with more than one type of volume (e.g. EBS and Portworx), if the user only configures a VolumeSnapshotLocationconfigured for EBS, then Velero will only snapshot the EBS volumes.

5. Restic data is stored under a prefix/subdirectory of the main Velero bucket, and will go into the bucket corresponding to the BackupStorageLocation selected by the user at backup creation time.

## Outlook

Velero's agnostic posture is on close watch as an Open source project.

When assessing the outlook of an open source project it is important to evaluate on 3 key factors:
1. Ability to nimbly incorporate features benefiting the community
2. Diversity in contributors and number of active contributions 
2. Adoption that drives the need for new features functions as additional usecases are identified. 


1. Incorporating Community Features

The single most important advancement in Velero is the adoption of Container Storage Interface (CSI). [CSI was developed and made GA in December of 2018 as a standard as of Kubernetes v1.13](https://kubernetes.io/blog/2018/12/03/kubernetes-1-13-release-announcement/) for exposing arbitrary block and file storage storage systems to containerized workloads on Container Orchestration Systems (COs) like Kubernetes. Prior to this, third-party storage code caused reliability and security issues in core Kubernetes binaries and the code was often difficult (and in some cases impossible) for Kubernetes maintainers to test and maintain. With the adoption of the Container Storage Interface, the Kubernetes volume layer becomes truly extensible. Using CSI, third-party storage providers can write and deploy plugins exposing new storage systems in Kubernetes without ever having to touch the core Kubernetes code. This gives Kubernetes users more options for storage and makes the system more secure and reliable.

Shortly after the acquisition of Heptio by VMware, Velero was born as a project. Largely driven by core-contributors from Heptio the community saw early investment in driving tighter integration with VMware.

Features such as support for backup and restore of Stateful [applications running natively on vSphere](https://velero.io/blog/velero-v1-1-stateful-backup-vsphere/) were prioritized over CSI in v1.1.

Furthur, integration into [VMware Tanzu portfolio](https://velero.io/blog/announcing-gh-move/)  in v1.2 was prioritized over much awaited CSI capabilities which is still in beta.

VMware continues to drive proliferation of Velero in its EMC storage family of products. For example Velero is gaining usability features starting with its [deep integration in PowerProtect](https://itzikr.wordpress.com/2019/12/31/dell-emc-powerprotect-19-3-is-available-kubernetes-integration-you-bet/)

Velero's implementation of CSI requires Kubernetes version 1.17 or greater and greatly relies on adoption and testing on the [three main hyperscalar Cloud Service Providers that Velero formally supports](https://velero.io/docs/master/supported-providers/). As a result, the much awaited CSI capability continues to remain in beta. Velero's limited support stance coupled with slow adoption by Cloud Providers on upstream Kubernetes releases is hampering innovation.

As of writing this article, the commercial Cloud Providers list that can realistically test CSI features of Velero are:
Kubernetes Service                  Latest K8s version supported      
IBM Kubernetes Service (IKS)    [1.17](https://cloud.ibm.com/docs/containers?topic=containers-cs_versions)               
and following commercial Cloud Providers list that cannot realistically test CSI features of Velero are:
Amazon (EKS)                            [1.15.11](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
Azure (AKS)                             [1.15.x](https://azure.microsoft.com/en-us/updates/azure-kubernetes-service-will-be-retiring-support-for-kubernetes-versions-1-11-and-1-12/)
Alibaba (CSK)                            [1.16.6](https://www.alibabacloud.com/help/doc-detail/115453.htm)
Google (GKE)                            [1.15.11](https://www.alibabacloud.com/help/doc-detail/115453.htm)
Oracle (CEK)                             [1.15.7](https://docs.cloud.oracle.com/en-us/iaas/releasenotes/changes/37013251-39b2-4c08-8536-906d76bba789/)

2. Contributors:
As of writing this document [4 of Velero contributors may be considered active and contributing](https://github.com/vmware-tanzu/velero/graphs/contributors)

3. Adoption:
As of writing this document [Velero has 10 adopters](https://github.com/vmware-tanzu/velero/blob/master/ADOPTERS.md)



## Summary

With these use cases, we have proven backup and restore of an application using Velero. Velero offers a developer friendly option to rapid recovery of container hosted applications and their supporting persistent volumes. The extensible plugin based model makes it possible for developers and administrators to support additional PersistentVolume types and Storage Classes. While, its strength lies in the Disaster Recovery space supporting Backup and Restore operations; it can support cluster portability by migrating resources between clusters e.g. between Dev/Test environments or across multiple cloud providers.



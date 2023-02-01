# Manage PDBs on Existing CDB

## Introduction

#### Managing PDBs on existing CDBs

In this lab we'll create an manage pluggable databases (**PDB's**) on an existing multitenant container database (**CDB**). A CDB includes zero, one, or many customer-created PDBs. A PDB is a portable collection of schemas, schema objects, and nonschema objects that appears to an Oracle Net client as a separate database.

The existing database could be running on Kubernetes or any other platform outside of Kubernetes. In this Lab, we will use the database provisioned in Kubernetes in the previous Lab. 

In order to be able to handle PDBs in an existing database remotely using the Kubernetes API, first we will have to bind to that database. Oracle database Operator uses Oracle Rest Data Services (**ORDS**) to do it. ORDS enables database management via API rest, including PDB management.

Estimated Lab Time: 20 minutes



## Task 1: Prepare CDB for PDB Lifecycle Management (PDB-LM)

Pluggable Database management operation is performed in the Container Database (CDB) and it includes create, clone, plug, unplug, delete, modify and map operations.

You cannot have an ORDS enabled schema in the container database. To perform the PDB lifecycle management operations, the default CDB administrator credentials must be defined by performing the below steps on the target CDB(s):

Create the CDB administrator user and grant the required privileges. In this lab, the user is C##DBAPI_CDB_ADMIN


1. In the SQLPlus session you opened in the previous Lab, run the follow commands editing the password for the C##DBAPI_CDB_ADMIN user:

**Tip**: We will use multiple credentials in this setup. You can set always the same pwd for all the users in this lab in order to simplify it and avoid using wrong pwd in the wrong place. Of course, it is not recommended for production setups. 


   ```
   -- Create the below users at the database level:
      
   ALTER SESSION SET "_oracle_script"=true;
   DROP USER  C##DBAPI_CDB_ADMIN cascade;
   CREATE USER C##DBAPI_CDB_ADMIN IDENTIFIED BY <Password> CONTAINER=ALL ACCOUNT UNLOCK;
   GRANT SYSOPER TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   GRANT SYSDBA TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   GRANT CREATE SESSION TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   
   -- Verify the account status of the below usernames. They should not be in locked status:

   col username        for a30
   col account_status  for a30
   select username, account_status from dba_users where username in ('ORDS_PUBLIC_USER','C##DBAPI_CDB_ADMIN','APEX_PUBLIC_USER','APEX_REST_PUBLIC_USER');
   ```

2. Now we need to open PDB$SEED in RW mode:

   ```
   ALTER PLUGGABLE DATABASE PDB$SEED CLOSE;
   ALTER PLUGGABLE DATABASE PDB$SEED OPEN READ WRITE;
   ```

   You can run the follow command to check existing PDBs status:
   ```
   Show pdbs;
   ```

Now, the CDB is ready to manage PDBs using ORDS Rest API


## Task 2: Create the ORDS image 

Oracle DB Operator Multitenant Database controller requires the Oracle REST Data Services (ORDS) image for PDB Lifecycle Management in the target CDB

In the below steps, we are building an ORDS Docker Image for ORDS Software. The image built can be later pushed to a local repository to be used later for a deployment.


1. Create a Developer Compute Instance

Due to OCI CLI limitations, we can not use it to create the image. We need to create a compute instance using Cloud Developer Image (Git, Docker and other required tools are ready using this image)

** NOTE **: Describe how to create the compute instance and how to login.

2. Clone the operator project using git

```
git clone https://github.com/oracle/oracle-database-operator.git

Cloning into 'oracle-database-operator'...
remote: Enumerating objects: 5447, done.
remote: Counting objects: 100% (1294/1294), done.
remote: Compressing objects: 100% (198/198), done.
remote: Total 5447 (delta 1182), reused 1096 (delta 1096), pack-reused 4153
Receiving objects: 100% (5447/5447), 3.25 MiB | 16.41 MiB/s, done.
Resolving deltas: 100% (3544/3544), done.
```
The Dockerfile and addional required files for ORDS image creation are available in the ORDS folder:

```
cd oracle-database-operator/ords/
```

2. Login into the required Docker Registries 

We will use an Java Docker image available in the Oracle Public Container Registry. 
We will need to the URL https://container-registry.oracle.com , Sign in, then click on "Java" and then accept the agreement.

Now, we can login in the 

```
docker login container-registry.oracle.com
```
It will ask for your user and pwd. Remember, you must use your Oracle web site credentials.

We are going to create our own ORDS docker image, and we will poll it to a private repo. We will use OCI Container Registry. 

** NOTE** : Describe how to login into OCI restry from https://www.oracle.com/webfolder/technetwork/tutorials/obe/oci/registry/index.html


3. Build ORDS Docker Image 

Run below command:

```
docker build -t oracle/ords-dboper:ords-latest .
```

You can check the docker image details using:

```
docker images
```

4. Push ORDS Docker Image to the OCI Repository

First, we need to tag the image running bellow command, setting your OCI region and repo_name:

```
docker tag oracle/ords-dboper:ords-latest <region-key>.io/<repo_name>/oracle/ords:latest
```

Now, you can push the image to the OCI repository:

```
docker push <region-key>.io/<repo_name>/oracle/ords:latest
```

Now, the Docker Image is ready to be use, but we still need some extra configurations.
We will use this image from Kubernetes, so we need to create a Kubernetes Secret for our OCI registry.
To do it, we will back to the OCI CLI and login from there to the OCI registry in the same way we did in the compute:

```
docker login <region-key>.io
```
Provide your OCI user and auth token, and then we are ready to create the Kubernetes Secret running:

```
kubectl create secret generic container-registry-secret --from-file=.dockerconfigjson=./.docker/config.json --type=kubernetes.io/dockerconfigjson -n oracle-database-operator-system
```
This Kubernetes secret will be provided in the .yaml file against the parameter `ordsImagePullSecret` to pull the ORDS Docker Image from your docker repository (if its a private repository).

## Task 3: Create the Kubernetes Secrets

Oracle DB Operator Multitenant Database Controller uses Kubernetes Secrets to store usernames and passwords to manage the life cycle operations of a PDB in the target CDB.

We will create two Kubernetes secrets; one required to connect to the CDB to perform the life cycle operations, and the second one with the sys credentials for the PDB. 

All credentials (user name and password) in both secrets must be base64 encoded. In order to get the base64 encoded value for a string, please use the below command like below at the command prompt. The value you get is the base64 encoded value for that string.

In addition to that, in order to use https protocol, all certificates need to be stored using Kubernetes Secret.

```
echo -n "<string to be encoded using base64>" | base64
```

1. Create the CDB Kubernetes Secret: 

This Secret will be use to create and configure the ORDS pod we will use later for the life cycle operations. 
For this lab we'll use the file [cdb_secret.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/cdb_secret.yaml):

```
#
# Copyright (c) 2022, Oracle and/or its affiliates. All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.
#
apiVersion: v1
kind: Secret
metadata:
  name: cdb1-secret
  namespace: oracle-database-operator-system
type: Opaque 
data: 
  ords_pwd: "d2VsY29tZTE=" 
  sysadmin_pwd: "V0VsY29tZV8xMiMj"
  cdbadmin_user: "QyMjREJBUElfQ0RCX0FETUlO"
  cdbadmin_pwd: "V0VsY29tZV8xMiMj"
  webserver_user: "c3FsX2FkbWlu"
  webserver_pwd: "d2VsY29tZTE="
```
sysadmin_pwd must be the sys pwd you set during the database creation in previous lab
cdbadmin_user and cdbadmin_pwd has been set on Task 1 for this lab. 

Use default values for ords_pwd, webserver_user and webserver_pwd

You can use the follow command to decode the credentials in this file:

```
echo -n '<String to be decoded>' | base64 --decode
```

And replace them with your custom credentials base64 encoded

```
echo -n "<string to be encoded using base64>" | base64
```

2. Create the PDB Kubernetes Secret:

For this lab we'll use the file [pdb_secret.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/pdb_secret.yaml):

```
#
# Copyright (c) 2022, Oracle and/or its affiliates. All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.
#
apiVersion: v1
kind: Secret
metadata:
  name: pdb1-secret
  namespace: oracle-database-operator-system
type: Opaque 
data: 
  sysadmin_user: "cGRiYWRtaW4="
  sysadmin_pwd: "V0VsY29tZV8xMiMj"
```

3. Create the Kubernetes Secret for Certificates

Run the follow commands to create certificates and key on your local host:

```
openssl genrsa -out ca.key 2048    
openssl req -new -x509 -days 365 -key ca.key -subj "/C=CN/ST=GD/L=SZ/O=oracle, Inc./CN=oracle Root CA" -out ca.crt
openssl req -newkey rsa:2048 -nodes -keyout tls.key -subj "/C=CN/ST=GD/L=SZ/O=oracle, Inc./CN=cdb-dev-ords" -out server.csr
/usr/bin/echo "subjectAltName=DNS:cdb-dev-ords,DNS:www.example.com" > extfile.txt
openssl x509 -req -extfile extfile.txt -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt
```
Now use them to create Kubernetes Secret:

```
kubectl create secret tls db-tls --key="tls.key" --cert="tls.crt"  -n oracle-database-operator-system
kubectl create secret generic db-ca --from-file=ca.crt -n oracle-database-operator-system
```

**Note**: On successful creation of the certificates secret creation remove files or move to secure storage.

## Task 4: Create the Kubernetes Custom Resource Definition for the CDB
The Oracle Database Operator Multitenant Controller creates the CDB kind as a custom resource that models a target CDB as a native Kubernetes object. This is used only to create Pods to connect to the target CDB to perform PDB-LM operations. These CDB resources can be scaled up and down based on the expected load using replicas

To create a CDB CRD, use the file [create_cdb.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/create_cdb.yaml). Let's see this file:

We will create the CRD "cdb-dev" on the namespace "oracle-database-operator-system"

```
#
# Copyright (c) 2022, Oracle and/or its affiliates. All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.
#
apiVersion: database.oracle.com/v1alpha1
kind: CDB
metadata:
  name: cdb-dev
  namespace: oracle-database-operator-system
```
In the specifications, we will provide the connection details for the CDB we want to bind for life cycle operations. Scan name is used only in case we connect to an Oracle RAC. We will set it with the same value than scanName for non-RAC setups.

In order to connect to your database running on Kubernetes, you can get the connection details running:

```
kubectl get singleinstancedatabase

NAME         EDITION      STATUS    VERSION      CONNECT STR                TCPS CONNECT STR   OEM EXPRESS URL
sidb-test1   Enterprise   Healthy   21.3.0.0.0   141.147.41.68:1521/ORCL1   Unavailable        https://141.147.41.68:5500/em
```


Finally, we provide the details for the ORDS configuration providing the image details and setting the credentials using the kubernetes secret created in the previous tasks:

```
spec:
  cdbName: "DemoDB_dev"
  scanName: "141.147.41.68"
  dbServer: "141.147.41.68"
  ordsImage: fra.ocir.io/frvqpcm8utic/oracle/ords:21.4.3
  ordsImagePullSecret: "container-registry-secret"
  dbPort: 1521
  replicas: 1
  serviceName: "DemoDB_dev.sub08311727040.dbcs4k8svcn.oraclevcn.com"
  sysAdminPwd:
    secret:
      secretName: "cdb-secret"
      key: "sysadmin_pwd"
  ordsPwd:
    secret:
      secretName: "cdb-secret"
      key: "ords_pwd"
  cdbAdminUser:
    secret:
      secretName: "cdb-secret"
      key: "cdbadmin_user"
  cdbAdminPwd:
    secret:
      secretName: "cdb-secret"
      key: "cdbadmin_pwd"
  webServerUser:
    secret:
      secretName: "cdb-secret"
      key: "webserver_user"
  webServerPwd:
    secret:
      secretName: "cdb-secret"
      key: "webserver_pwd"
```

Apply this configuration:

```
kubectl apply -f cdb-dev.yaml
```

You can list the pods in the oracle-database-operator-system. It should show the pod for this CDB.

## Task 5: Manage PDBs on the configured CDB

The Oracle Database Operator On-Prem Controller creates the PDB kind as a custom resource that models a PDB as a native Kubernetes object. There is a one-to-one mapping between the actual PDB and the Kubernetes PDB Custom Resource.

Using Oracle DB Operator Multitenant Controller, you can perform the following PDB-LM operations: CREATE, CLONE, MODIFY, DELETE, UNPLUG, PLUG.

1. Create a PDB

To create a PDB CRD, use the file [create_pdb.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/create_pdb.yaml). Let's see this file:

We are going to create a PDB CRD called pdb1 in oracle-database-operator-system namespace. We are adding a label in order to know which CDB we will use:

```
#
# Copyright (c) 2022, Oracle and/or its affiliates. All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.
#
apiVersion: database.oracle.com/v1alpha1
kind: PDB
metadata:
  name: pdb1
  namespace: oracle-database-operator-system
  labels:
    cdb: cdb-dev
```

We will specify the CDB resource name and the cdbName we set into the CDB custom resource definition:

```
spec:
  cdbResName: "cdb-dev"
  cdbName: "DemoDB_dev"
  pdbName: "pdbnew"
```

And we will provide the admin name and password from the kubernetes secret we created in previous tasks:

```
  adminName:
    secret:
      secretName: "pdb1-secret"
      key: "sysadmin_user"
  adminPwd:
    secret:
      secretName: "pdb1-secret"
      key: "sysadmin_pwd"
```
We add some database configuration:

```
  fileNameConversions: "NONE"
  totalSize: "1G"
  tempSize: "100M"
```
And we set the action to **create**

```
  action: "Create"
```

Now, we can apply the yaml file:

```
kubetl apply -f create_pdb.yaml
```

The PDB should be created in the CDB. Let check it. We can use our SQLPlus connection to the CDB to list the PDBs:

```
Show pdbs;
```

Or we can use kubectl to list the PDBs managed by the operator:

```
kubectl get pdbs -n oracle-database-operator-system
```

Our PDB1 has been created and it is open in read/write mode.

2. Clone a PDB

To clone a PDB from an existing one, use the file [clone_pdb.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/clone_pdb.yaml).

There are some differences with the create_pdb.yaml file:
- We need to set the source pdb: srcPdbName
- We don't set the admin credentials because the clone will have the same ones than the source.
- The action has changed to CLONE

```
#
# Copyright (c) 2022, Oracle and/or its affiliates. All rights reserved.
# Licensed under the Universal Permissive License v 1.0 as shown at http://oss.oracle.com/licenses/upl.
#
apiVersion: database.oracle.com/v1alpha1
kind: PDB
metadata:
  name: pdb1-clone
  namespace: oracle-database-operator-system
  labels:
    cdb: cdb-dev
spec:
  cdbResName: "cdb-dev"
  cdbName: "DemoDB_dev"
  pdbName: "pdbnewclone"
  srcPdbName: "pdbnew"
  fileNameConversions: "NONE"
  totalSize: "UNLIMITED"
  tempSize: "UNLIMITED"
  action: "Clone"
```
Now, you can apply this CDR:

```
kubectl apply -f clone_pdb.yaml
```

Now, you can check if the clone has been created using your SQLPlus connection:

```
Show pdbs;
```

You can use kubectl to list those PDBs managed by the operator:
```
kubectl get pdbs -n oracle-database-operator-system
```

3. Delete the cloned PDB

We are going to delete the clone we have created in the previous step. We will use the file [delete_clone_pdb.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/delete_clone_pdb.yaml).

4. Close and Unplug PDB

You can disassociate or unplug a PDB from a CDB and reassociate or plug the PDB into the same CDB or into another CDB.

In order to unplug a PDB, It must be closed for R/W operations. We will close the PDB we created using the file [close_pdb.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/close_pdb.yaml).

You can use your SQLPlus connection or kubectl get pdbs command to check the PDB status. It should be closed now.

Now, we can proceed to unplug the PDB from the CDB. We will use the file [unplug_pdb.yaml](https://github.com/oracle-livelabs/developer/blob/main/db-operator-k8s/manage-pdbs/files/unplug_pdb.yaml).

Now, you can check the PDBs available on the CDB. PDB1 should not be there. 




Congratulations, your database is up and running, and you are able to connect to it through Enterprise Manager and Sqlplus !  You may now **proceed to the next lab**.

## Acknowledgements
* **Author** - Victor Mendo, December 2022
* **Last Updated By/Date** - Victor Mendo, December 2022 

<span style="font-family:Liberation mono; font-size:0.8em; line-height: 1.2em">



# Oracle Multitenant Database Controller

CDBs and PDBs are the part of the Oracle Database's [Multitenant Architecture](https://docs.oracle.com/en/database/oracle/oracle-database/21/multi/introduction-to-the-multitenant-architecture.html#GUID-AB84D6C9-4BBE-4D36-992F-2BB85739329F). The Multitenant Database Controller is a feature of Oracle DB Operator for Kubernetes (OraOperator) which helps to manage the life cycle of Pluggable Databases (PDBs) in an Oracle Container Database(CDB).

The target CDB (for which the PDB life cycle management is needed) can be running on an multitenant machine and to manage its PDBs, the Oracle DB Operator can run on an multitenant Kubernetes system (For Example: [Oracle Linux Cloud Native Environment or OLCNE](https://docs.oracle.com/en/operating-systems/olcne/)).

NOTE: The target CDB (for which the PDB life cycle management is needed) **can also** run in a Cloud environment as well (For Example: OCI's [Oracle Base Database Service](https://docs.oracle.com/en-us/iaas/dbcs/doc/bare-metal-and-virtual-machine-db-systems.html)) and to manage its PDBs, the Oracle DB Operator can run on a Kubernetes Cluster running in cloud (For Example: OCI's [Container Engine for Kubernetes or OKE](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengoverview.htm#Overview_of_Container_Engine_for_Kubernetes))



# Oracle DB Operator Multitenant Database Controller Deployment

To deploy OraOperator, use this [Oracle Database Operator for Kubernetes](https://github.com/oracle/oracle-database-operator/blob/main/README.md) step-by-step procedure.

After the Oracle Database Operator is deployed, you can see the DB Operator Pods running in the Kubernetes Cluster. As part of the OraOperator deployment, the multitenant Database Controller is deployed and we can see the CRDs (Custom Resource Definition) for CDB and PDB in the list of CRDs. The following output is an example of such a deployment:
```bash
[root@test-server oracle-database-operator]# kubectl get ns
NAME                              STATUS   AGE
cert-manager                      Active   32h
default                           Active   245d
kube-node-lease                   Active   245d
kube-public                       Active   245d
kube-system                       Active   245d
oracle-database-operator-system   Active   24h    <<<< namespace to deploy the Oracle Database Operator


[root@test-server oracle-database-operator]# kubectl get all -n  oracle-database-operator-system
NAME                                                               READY   STATUS    RESTARTS   AGE
pod/oracle-database-operator-controller-manager-665874bd57-dlhls   1/1     Running   0          28s
pod/oracle-database-operator-controller-manager-665874bd57-g2cgw   1/1     Running   0          28s
pod/oracle-database-operator-controller-manager-665874bd57-q42f8   1/1     Running   0          28s

NAME                                                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/oracle-database-operator-controller-manager-metrics-service   ClusterIP   10.96.130.124   <none>        8443/TCP   29s
service/oracle-database-operator-webhook-service                      ClusterIP   10.96.4.104     <none>        443/TCP    29s

NAME                                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/oracle-database-operator-controller-manager   3/3     3            3           29s

NAME                                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/oracle-database-operator-controller-manager-665874bd57   3         3         3       29s
[root@docker-test-server oracle-database-operator]# 


[root@test-server oracle-database-operator]# kubectl get crd
NAME                                               CREATED AT
autonomouscontainerdatabases.database.oracle.com   2022-06-22T01:21:36Z
autonomousdatabasebackups.database.oracle.com      2022-06-22T01:21:36Z
autonomousdatabaserestores.database.oracle.com     2022-06-22T01:21:37Z
autonomousdatabases.database.oracle.com            2022-06-22T01:21:37Z
cdbs.database.oracle.com                           2022-06-22T01:21:37Z <<<<
certificaterequests.cert-manager.io                2022-06-21T17:03:46Z
certificates.cert-manager.io                       2022-06-21T17:03:47Z
challenges.acme.cert-manager.io                    2022-06-21T17:03:47Z
clusterissuers.cert-manager.io                     2022-06-21T17:03:48Z
dbcssystems.database.oracle.com                    2022-06-22T01:21:38Z
issuers.cert-manager.io                            2022-06-21T17:03:49Z
oraclerestdataservices.database.oracle.com         2022-06-22T01:21:38Z
orders.acme.cert-manager.io                        2022-06-21T17:03:49Z 
pdbs.database.oracle.com                           2022-06-22T01:21:39Z <<<<
shardingdatabases.database.oracle.com              2022-06-22T01:21:39Z
singleinstancedatabases.database.oracle.com        2022-06-22T01:21:40Z
```

The following sections explain the setup and functionality of this controller.


# Prerequsites to manage PDB Life Cycle using Oracle DB Operator Multitenant Database Controller

Before you want to manage the life cycle of a PDB in a CDB using the Oracle DB Operator Multitenant Database Controller, complete the following steps.

**CAUTION :** You must make the changes specified in this section before you proceed to the next section.

* [Prepare CDB for PDB Lifecycle Management or PDB-LM](#prepare-cdb-for-pdb-lifecycle-management-pdb-lm)
* [Oracle REST Data Service or ORDS Image](#oracle-rest-data-service-ords-image)
* [Kubernetes Secrets](#kubernetes-secrets)
* [Kubernetes CRD for CDB](#kubernetes-crd-for-cdb)
* [Kubernetes CRD for PDB](#kubernetes-crd-for-pdb)


+ ## Prepare CDB for PDB Lifecycle Management (PDB-LM)

Pluggable Database management operation is performed in the Container Database (CDB) and it includes create, clone, plug, unplug, delete, modify and map operations.

You cannot have an ORDS enabled schema in the container database. To perform the PDB lifecycle management operations, the default CDB administrator credentials must be defined by performing the below steps on the target CDB(s):

Create the CDB administrator user and grant the required privileges. In this example, the user is `C##DBAPI_CDB_ADMIN`. However, any suitable common user name can be used.

```SQL
SQL> conn /as sysdba
 
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

### Reference Setup: Example of a setup using OCI OKE(Kubernetes Cluster) and a CDB in Cloud (OCI Exadata Database Cluster)

Please refer [here](./provisioning/example_setup_using_oci_oke_cluster.md) for steps to configure a Kubernetes Cluster and a CDB. This example uses an OCI OKE Cluster as the Kubernetes Cluster and a CDB in OCI Exadata Database service. 

+ ## Oracle REST Data Service (ORDS) Image

  Oracle DB Operator Multitenant Database controller requires the Oracle REST Data Services (ORDS) image for PDB Lifecycle Management in the target CDB. 
  
  You can build this image by using the ORDS [Dockerfile](../../../ords/Dockerfile)
  
  > **_NOTE:_** The current version of Oracle DB Operator Multitenant Controller has been tested with **ords 22.2.1.202.1302** version.

Please refer [here](./provisioning/ords_image.md) for the steps to build ORDS Docker Image with  **ords 22.2.1.202.1302** version.


+ ## Kubernetes Secrets

  Oracle DB Operator Multitenant Database Controller uses Kubernetes Secrets to store usernames and passwords to manage the life cycle operations of a PDB in the target CDB. In addition to that ,in order to use https protocol, all certificates need to be stored using Kubernests Secret. 

### Secrets for CDB CRD

  Create a secret file as shown here: [config/samples/multitenant/cdb_secret.yaml](../../config/samples/multitenant/cdb_secret.yaml). Modify this file with the `base64` encoded values of the required passwords for CDB and use it to create the required secrets.

  ```bash
  kubectl apply -f cdb_secret.yaml
  ```
  
  **Note:** In order to get the base64 encoded value for a password, please use the below command like below at the command prompt. The value you get is the base64 encoded value for that password string.

  ```bash
  echo -n "<password to be encoded using base64>" | base64
  ```

  **Note:** <span style="color:red">  On successful creation of the CDB Resource, the CDB secrets would be deleted from the Kubernetes </span> .

### Secrets for PDB CRD
  Create a secret file as shown here: [config/samples/multitenant/pdb_secret.yaml](../../config/samples/multitenant/pdb_secret.yaml). Modify this file with the `base64` encoded values of the required passwords for PDB and use it to create the required secrets. 

  ```bash
  kubectl apply -f pdb_secret.yaml
  ```
  **NOTE:** Refer to command provided above to encode the password using base64.
  
  **NOTE:** <span style="color:red"> Don't leave plaintext files containing sensitive data on disk. After loading the Secret, remove the plaintext file or move it to secure storage. </span>
  
### Secrets for CERTIFICATES

Create certificates and key on your local host and use them to create Kubernet secret

```bash
genrsa -out ca.key 2048
openssl req -new -x509 -days 365 -key ca.key -subj "/C=CN/ST=GD/L=SZ/O=oracle, Inc./CN=oracle Root CA" -out ca.crt
openssl req -newkey rsa:2048 -nodes -keyout tls.key -subj "/C=CN/ST=GD/L=SZ/O=oracle, Inc./CN=cdb-dev-ords" -out server.csr
/usr/bin/echo "subjectAltName=DNS:cdb-dev-ords,DNS:www.example.com" > extfile.txt
openssl x509 -req -extfile extfile.txt -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt
```

```bash
kubectl create secret tls db-tls --key="tls.key" --cert="tls.crt"  -n oracle-database-operator-system
kubectl create secret generic db-ca --from-file=ca.crt -n oracle-database-operator-system
```

<img src="openssl_schema.jpg" alt="image_not_found" width="900"/>
**Note:** <span style="color:red">  On successful creation of the certificates secret creation remove files or move to secure storage </span> .

+ ## Kubernetes CRD for CDB

The Oracle Database Operator Multitenant Controller creates the CDB kind as a custom resource that models a target CDB as a native Kubernetes object. This is  used only to create Pods to connect to the target CDB to perform PDB-LM operations. These CDB resources can be scaled up and down based on the expected load using replicas. Each CDB resource follows the CDB CRD as defined here: [config/crd/bases/database.oracle.com_cdbs.yaml](../../config/crd/bases/database.oracle.com_cdbs.yaml)

To create a CDB CRD, a sample .yaml file is available here: [config/samples/multitenant/cdb.yaml](../../config/samples/multitenant/cdb.yaml)

**Note:** The password and username fields in this *cdb.yaml* yaml are Kubernetes Secrets created earlier. Please see the section [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) for more information. Please see [Kubernetes Private Registry Documenation]( https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/) for creating secrets for pulling images from docker private registry.

1. [Use Case: Create a CDB CRD Resource](./provisioning/cdb_crd_resource.md)
2. [Use Case: Add another replica to an existing CDB CRD Resource](./provisioning/add_replica.md)


+ ## Kubernetes CRD for PDB

The Oracle Database Operator Multitenant Controller creates the PDB kind as a custom resource that models a PDB as a native Kubernetes object. There is a one-to-one mapping between the actual PDB and the Kubernetes PDB Custom Resource. You cannot have more than one Kubernetes resource for a target PDB. This PDB resource can be used to perform PDB-LM operations by specifying the action attribute in the PDB Specs. Each PDB resource follows the PDB CRD as defined here: [config/crd/bases/database.oracle.com_pdbs.yaml](../../../config/crd/bases/database.oracle.com_pdbs.yaml)

To create a PDB CRD Resource, a sample .yaml file is available here: [config/samples/multitenant/pdb_create.yaml](../../config/samples/multitenant/pdb_create.yaml)

# Use Cases for PDB Lifecycle Management Operations using Oracle DB Operator Multitenant Controller

Using Oracle DB Operator Multitenant Controller, you can perform the following PDB-LM operations: CREATE, CLONE, MODIFY, DELETE, UNPLUG, PLUG.

1. [Create PDB](./provisioning/create_pdb.md)
2. [Clone PDB](./provisioning/clone_pdb.md)
3. [Modify PDB](./provisioning/modify_pdb.md)
4. [Delete PDB](./provisioning/delete_pdb.md)
5. [Unplug PDB](./provisioning/unplug_pdb.md)
6. [Plug PDB](./provisioning/plug_pdb.md)


## Validation and Errors

Please check [here](./provisioning/validation_error.md) for the details to look for any validation error.


## Known issues

Please refer [here](./provisioning/known_issues.md) for the known issues related to Oracle DB Operator Multitenant Controller.

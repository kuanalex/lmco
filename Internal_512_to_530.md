# Upgrade Runbook - v.5.1.2 to 5.3.0

---

## Upgrade documentation

[Upgrading from IBM Cloud Pak for Data Version 5.1.x to Version 5.3.x](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading-from-version-51)

## Upgrade context

From

```
OCP: Self managed AWS 4.17
CPD: 5.1.2
Storage:  Netapp 
Componenets: cpd_platform,scheduler,wkc,mantaflow,analyticsengine,datastage_ent,datastage_ent_plus,dv,dp,db2oltp,ws,ws_runtimes,wml,ws_pipelines,dmc,db2wh,rstudio,spss,cognos_analytics,planning_analytics
```

To

```
OCP:Self managed 4.17 
CPD: 5.3.1
Storage:Netapp 
Componenets: cpd_platform,scheduler,wkc,mantaflow,analyticsengine,datastage_ent,datastage_ent_plus,dv,dp,db2oltp,ws,ws_runtimes,wml,ws_pipelines,dmc,db2wh,rstudio,spss,cognos_analytics,planning_analytics
```

## Pre-requisites

#### 1. Backup of the cluster is done.

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicate approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry.
`<br>`
Reference:
[Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

##### Mirroring Images for Redhat Certificate Manager

* Set your Cluster configguration directory

  ```bash
  CLUSTER_CONFIG_DIRS=<REPLACE WITH DESIRED PATH>
  ```
* Set your OCP version

```bash
  export OPENSHIFT_VERSION=v4.17
```

- Create an ImageSetConfigutation named red-hat-cert-manager.yaml

  ```bash
  cat << EOF > ./red-hat-cert-manager.yaml
  kind: ImageSetConfiguration
  apiVersion: mirror.openshift.io/v2alpha1
  mirror:
    operators:
      - catalog: registry.redhat.io/redhat/redhat-operator-index:${OPENSHIFT_VERSION}
        packages: 
        - name: openshift-cert-manager-operator
  EOF
  ```
- Mirror the Images

  ```bash
  oc mirror \
  -c ret-hat-cert-manager.yaml \
  --workspace file://${CLUSTER_CONFIGS_DIR} docker://${PRIVATE_REGISTRY_LOCATION} \
  --v2
  ```
- Apply the yamls

  ```bash
  oc apply -f ${CLUSTER_CONFIGS_DIR}/working-dir/cluster-resources
  ```

  ```bash
  oc apply -f working-dir/cluster-resources/signature-configmap.json
  ```

  Verify ImageTagMirrorSet is created and Catalogsource

  ```bash
  oc get imagedigestmirrorset
  ```

  ```bash
  oc get catalogsource -n openshift-marketplace
  ```

#### 3. The permissions required for the upgrade is ready

- Openshift cluster administrator permissions
- Cloud Pak for Data administrator permissions
- Permission to access the private image registry for pushing or pull images
- Access to the Bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade.

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status.

#### 5. Backup the Routes

```
oc get Routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > Routes_Bak.yaml
```

## Table of Content

```
Part 1: Pre-upgrade
1.1 Update client workstation
1.1.1 Set up the utilities
1.1.2 Update environment variables
1.1.3 Ensure the cpd-cli manage plug-in has the latest version of the olm-utils image
1.1.4 Download CASE packages
1.1.5 Create a profile for upgrading the service instances
1.2 Health check OCP & CPD

Part 2: Upgrade
2.1 Upgrade CPD to 5.3.0
2.1.1 Upgrade shared cluster components
2.1.2 Prepare to upgrade IBM Software Hub
2.1.3 Create image pull secrets for IBM Software Hub instance
2.1.4 Upgrade IBM Software Hub
2.2 Upgrade watsonx Orchestrate
2.2.1 Specify the parameters in the `install-options.yml` file
2.2.2 Upgrade the watsonx Orchestrate service
2.2.3 Apply the hot fix
2.2 Upgrade watsonx.ai services
2.3 Upgrade watsonx.governance services
2.4 Upgrade watson Speech
2.5 Upgrade Voice Gateway
2.6 Upgade CPD BR service

Summarize and close out the upgrade

```

## Part 1: Pre-upgrade

### 1.1 Update client workstation

#### 1.1.1 Set up the utilities

**1. Update the cpd-cli utility**

<br>

Update the cpd-cli utility following the steps in [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=52-updating-client-workstations)

<br>

After the update is done, run below commands for the confirmation:

```
cpd-cli version
```

Output like this

```
cpd-cli
	Version: 14.3.0
	Build Date: 2025-12-05T14:54:42
	Build Number: 2792
	SWH Release Version: 5.3.0
```

**2. Update the OpenShift CLI**
`<br>`
Check the OpenShift CLI version.

```
oc version
```

If the version doesn't match the OpenShift cluster version, update it accordingly.

**3. Install the Helm CLI**
`<br>`
Install Helm by following the [Helm documentation](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

#### 1.1.2 Update environment variables

Make a copy of the environment variables script used by the existing 5.2.2 instance with the name like `cpd_vars_530.sh`.
`<br>`
Update the environment variables script `cpd_vars_530.sh` as follows.

```
vi cpd_vars_530.sh
```

1.Locate the VERSION entry and update the environment variable for VERSION.

```
export VERSION=5.3.0
```

2.Locate the COMPONENTS entry and confirm the COMPONENTS entry is accurate.

```
export COMPONENTS=cpd_platform,scheduler,wkc,mantaflow,analyticsengine,datastage_ent,datastage_ent_plus,dv,dp,db2oltp,ws,ws_runtimes,wml,ws_pipelines,dmc,db2wh,rstudio,spss,cognos_analytics,planning_analytics
```

3.Add a new section called Image pull configuration to your script and add the following environment variables

```
export IMAGE_PULL_SECRET=ibm-entitlement-key
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
export IMAGE_PULL_PREFIX=${PRIVATE_REGISTRY_LOCATION}
```

Save the changes. `<br>`

Confirm that the script does not contain any errors.

```
bash ./cpd_vars_530.sh
```

Run this command to apply cpd_vars_530.sh

```
source cpd_vars_530.sh
```

#### 1.1.3 Ensure the cpd-cli manage plug-in has the latest olm-utils image

```
cpd-cli manage restart-container
```

**Note:**
`<br>`Check and confirm the olm-utils-v4 container is up and running.

```
podman ps | grep olm-utils-v4
```

#### 1.1.4 Downloading CASE packages

Downloading CASE packages before running IBM Software Hub upgrade commands.

**Note:** If the CASE packages have already been downloaded when mirroring the images, this step can be skipped.

<br>

[Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-1)

#### 1.1.5 Creating a profile for upgrading the service instances

Create a profile on the workstation from which you will upgrade the service instances.

The profile must be associated with a Cloud Pak for Data user who has either the following permissions:

- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

Click this link and follow these steps for getting it done.

[Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

#### 1.1.6 Capturing CCS Storage migration

- Export your CPD url

  ```bash
  export INSTANCE_URL=<URL>
  ```
- Retrieve WDP Credentials

  ```bash
  TOKEN=$(oc get -n ${PROJECT_CPD_INST_OPERANDS} secrets wdp-service-id -o yaml | grep service-id-credentials | cut -d':' -f2- | sed -e 's/ //g' | base64 -d)
  ```
- Get Number of Catalogs, Projects, Spaces

  ```bash
  curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=catalogs&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
  ```

  ```bash
  curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=projects&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
  ```

  ```bash
  curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=spaces&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
  ```
- Create and run the prevalidation script

  ```bash
  vi precheck_migration.sh
  ```

  ```bash
  #!/bin/bash

  # Default ranges for couchdb size
  SMALL=50
  MEDIUM=100
  LARGE=200

  echo "Performing pre-migration checks"

  patch_for_small()
  {
    echo -e "Run the following command to increase the CPU and memory:\n"
            cat << EOF
  oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
    "catalog_api_postgres_migration_threads": 4,
    "catalog_api_migration_job_resources": { 
      "requests": {"cpu": "2", "ephemeral-storage": "10Mi", "memory": "2Gi"},
      "limits": {"cpu": "6", "ephemeral-storage": "1Gi", "memory": "6Gi"}}
  }}'
  EOF
      echo
      echo "The system is ready for migration. Upgrade your cluster as usual."
  }

  patch_for_medium()
  {
      echo -e "Run the following command to increase the CPU and memory:\n"
            cat << EOF
  oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
    "catalog_api_postgres_migration_threads": 6,
    "catalog_api_migration_job_resources": { 
      "requests": {"cpu": "3", "ephemeral-storage": "10Mi", "memory": "4Gi"},
      "limits": {"cpu": "8", "ephemeral-storage": "4Gi", "memory": "8Gi"}}
  }}'
  EOF
      echo
      echo "The system is ready for migration. Upgrade your cluster as usual."
  }

  patch_for_large()
  {
      echo -e "Run the following command to increase the CPU and memory:\n"
            cat << EOF
  oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
    "catalog_api_postgres_migration_threads": 8,
    "catalog_api_migration_job_resources": { 
      "requests": {"cpu": "6", "ephemeral-storage": "10Mi", "memory": "6Gi"},
      "limits": {"cpu": "10", "ephemeral-storage": "6Gi", "memory": "10Gi"}}
  }}'
  EOF
      echo
      echo "Before you can start the upgrade, you must prepare the system for migration."
  }

  check_resources(){
          scale_config=$1
          pvc_size=$(oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} database-storage-wdp-couchdb-0 --no-headers | awk '{print $4}')
          size=$(awk '{print substr($0, 1, length($0)-2)}' <<< "$pvc_size")

          if [[ $scale_config == "small" ]];then
            if [[ "$size" -le "$SMALL" ]];then
              echo "The system is ready for migration. Upgrade your cluster as usual."
            elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$MEDIUM" ];then
              patch_for_medium
            elif [ "$size" -ge "$MEDIUM" ] && [ "$size" -le "$LARGE" ];then
              patch_for_large
            else
              patch_for_large
            fi
          elif [[ $scale_config == "medium" ]];then
            if [[ "$size" -le "$SMALL" ]];then
              patch_for_small
            elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$MEDIUM" ];then
              echo "The system is ready for migration. Upgrade your cluster as usual."
            elif [ "$size" -ge "$MEDIUM" ] && [ "$size" -le "$LARGE" ];then
              patch_for_large
            else
              patch_for_large
            fi
          elif [[ $scale_config == "large" ]];then
            if [[ "$size" -le "$SMALL" ]];then
              patch_for_small
            elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$MEDIUM" ];then
              patch_for_medium
            elif [ "$size" -ge "$MEDIUM" ] && [ "$size" -le "$LARGE" ];then
              echo "The system is ready for migration. Upgrade your cluster as usual."
            else
              patch_for_large
            fi
          fi
  }

  check_upgrade_case(){   
          echo -e "Checking if automatic upgrade or semi-automatic upgrade is needed"
          scale_config=$(oc get ccs -n ${PROJECT_CPD_INST_OPERANDS} ccs-cr -o json | jq -r '.spec.scaleConfig')

          # Default case, scale config is set to small
          if [[ -z "${scale_config}" ]];then
            scale_config=small
          fi

          check_resources $scale_config
  }

  check_upgrade_case
  ```
- Capture the messages returned by script and the patch command into  css_migration.sh.

#### 1.1.7 Backing  up IBM Cert Manager for migration

- Create a new project
  ```bash
  oc new-project cert-mgr-test
  ```
- Back up the Issuer CR to a  file
  ```bash
  oc get issuers.cert-manager.io -A -o yaml > issuer_list.yaml
  ```
- Back up the Certificates
  ```bash
  oc get certificates.cert-manager.io -A -o yaml > certificate_list.yaml
  ```
- Verify IBM Cert manager is working as expected
  - Create the issuer yaml

    ```bash
    cat <<EOF |oc apply -f -
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: verify-issuer
    spec:
      selfSigned: {}
    EOF
    ```
  - Apply the issuer yaml

    ```bash
    oc apply -f issuer_list.yaml
    ```
  - Create the Certficate CR verify-certificate

    ```bash
    cat <<EOF |oc apply -f -
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: verify-certificate
    spec:
      commonName: verify-certificate
      secretName: verify-secret
      issuerRef:
        name: verify-issuer
        kind: Issuer
        group: cert-manager.io
    EOF
    ```
  - Apply the certififcate list yaml

    ```bash
    oc apply -f certificate_list.yaml
    ```
  - Veryify the CR is ready

    ```bash
    oc get issuers.cert-manager.io
    ```

### 1.2 Health check OCP & CPD

Check and make sure the cluster operators, nodes, and machine configure pool are in healthy status.
`<br>`
Log onto bastion node and then log into OCP.

```
${OC_LOGIN}
```

Check the status of nodes, cluster operators and machine config pool.

```
oc get nodes,co,mcp
```

2. Check Cloud Pak for Data status

Log onto bastion node, and make sure IBM Cloud Pak for Data command-line interface installed properly.

Run this command in terminal and make sure the Lite and all the services' status are in Ready status.

```
${CPDM_OC_LOGIN}
```

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Run this command and make sure all pods healthy.

```
oc get po --no-headers --all-namespaces -o wide | grep -Ev '([[:digit:]])/\1.*R' | grep -v 'Completed'
```

## Part 2: Upgrade

### 2.1 Upgrade CPD to 5.3.0

#### 2.1. Migrating to RedHat OpenShift Cert manager

##### Uninstalling IBM Cert Manager

- Get list of Certificate Manager Configuration

  ```
  oc get certmanagerconfig \
  -n ${PROJECT_CERT_MANAGER}
  ```
- Delete each configuration (Repeat for each name)

  ```BASH
  oc delete certmanagerconfig <name> \
  -n ${PROJECT_CERT_MANAGER}
  ```
- Uninstall the IBM Cert manager operator

  - Delete the subscription

    ```bash
    oc delete sub ibm-cert-manager-operator \
    -n ${PROJECT_CERT_MANAGER}
    ```
  - Get the CSV names

    ```bash
    oc get csv \
    -n ${PROJECT_CERT_MANAGER} \
    | grep ibm-cert-manager
    ```
  - Delete the CSV (Repeat for multiple CSVs)

    ```bash
    oc delete csv <name> \
    -n ${PROJECT_CERT_MANAGER}
    ```
- Verify All resources (Deployments, Service and mutating/validatingwebhooks)have been deleted, if not please proceed with deleting

  ```bash
  oc get deployments \
  -n ${PROJECT_CERT_MANAGER} \
  -l app.kubernetes.io/component=cert-manager
  ```

  ```bash
  oc get service \
  -n ${PROJECT_CERT_MANAGER} \
  -l app=ibm-cert-manager-webhook
  ```

  ```bash
  oc get mutatingwebhookconfiguration \
  -n ${PROJECT_CERT_MANAGER} \
  | grep cert-manager-webhook
  ```

  ```bash
   oc get validatingwebhookconfiguration \
  -n ${PROJECT_CERT_MANAGER} \
  | grep cert-manager-webhook
  ```

##### Installing RedHat Cert Manager

- Open OpenShift Webconsole UI
- Navigate to Operator -> Operator Hub
- Filter for term "cert-manager Operator for Red Hat OpenShift"
- Install the RH Cert manager operator

#### 2.1. Upgrade shared cluster components

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```
${CPDM_OC_LOGIN}
```

2.Upgrade the License Service.

Confirm the project in which the License Service is running.
`<br>`

```
oc get deployment -A |  grep ibm-licensing-operator
```

Make sure the project returned by the command matches the environment variable PROJECT_LICENSE_SERVICE in your environment variables script `cpd_vars_522.sh`.
`<br>`
Upgrade the License Service.

```bash
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Confirm that the License Service pods are Running or Completed:

```bash
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

#### 2.1.2 Prepare to upgrade IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster

```bash
${CPDM_OC_LOGIN}
```

2.Updating the cluster-scoped resources for the platform and services

```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

3. Change to the work directory. The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.

Log in to Red Hat® OpenShift® Container Platform as a cluster administrator.

```bash
${OC_LOGIN}
```

Apply the cluster-scoped resources for the from the `cluster_scoped_resources.yaml` file.

```bash
oc apply --server-side --force-conflicts -f cluster_scoped_resources.yaml
```

Have a record of the resources that you generated.

```bash
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml
```

Return to paretn folder.

3.Applying your entitlements to monitor and report use against license terms

**Production enironment (Id non-production, please set flag --production=false)**

Apply the IBM Cloud Pak for Data Enterprise Edition for the non-production environment.

```bash
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=true
```

Apply the Cognos license (Only Production license available).

```bash
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cognos-analytics
```

Apply the Datastage license.

```bash
export LICENSE_NAME=datastage

cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=${LICENSE_NAME} \
--production=true
```

Apply the Datastage Plus license.

```bash
export LICENSE_NAME=datastage-plus

cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=${LICENSE_NAME} \
--production=true
```

Apply the IKC/WKC Standard license (Only Production license available).

```bash
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=ikc-standard
```

Apply the Mantaflow license.

```bash
export LICENSE_NAME=data-lineage

cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=${LICENSE_NAME} \
--production=true
```

Apply the Planning Analytics licens (Only Production license available).

```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=planning-analytics
```

Reference: [Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-3)

#### 2.1.3 Create image pull secrets for IBM Software Hub instance

Log in to OpenShift cluster.

```bash
${OC_LOGIN}
```

Generate the image pull credentials:

```bash
export IMAGE_PULL_CREDENTIALS=$(echo -n "$PRIVATE_REGISTRY_PULL_USER:$PRIVATE_REGISTRY_PULL_PASSWORD" | base64 -w 0)
```

Create a file named dockerconfig.json based on where your cluster pulls images from:

```bash
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

Create the image pull secret in the `operators` project for the instance.

```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the `operands` project for the instance.

```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERANDS}
```

##### If there is Tethered namespace

```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INSTANCE_TETHERED}
```

#### 2.1.4 Upgrade IBM Software Hub

1.Run the cpd-cli manage login-to-ocp command to log in to the cluster.

```bash
${CPDM_OC_LOGIN}
```

2.Run the command in the ccs_migration.sh or run script directly.

```bash
./ccs_migration.sh
```

3.Upgrade the required operators and custom resources for the instance (For non-tethered namespace).

Note, if ether namespace is present please add flag "--tethered_instance_ns=${PROJECT_CPD_INSTANCE_TETHERED_LIST}"

```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Once the above command `cpd-cli manage install-components` complete, make sure the status of the IBM Software Hub is in 'Completed' status.

```bash
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

### 2.2 Upgrade CCS + WKC + Analytics Engine + Data Privacy

#### 2.2.1 Specify the parameters in the `override.yaml` file 

Specify the following options in the `override.yaml` file in the `work` directory. Create the `override.yaml` file if it doesn't exist in the `work` directory.

```bash
###############################################################################
```

**Note:** Make sure you edit or create the `override.yaml` file in the right `work` folder. You can identify the location of the `work` folder using below command.

```bash
podman inspect olm-utils-play-v4 | jq -r '.[0].Mounts' |jq -r '.[] | select(.Destination == "/tmp/work") | .Source'
```

```

```

#### 2.2.2 Upgrade the CCS + WKC + Analytics Engine + Data Privacy service.

- Log in to the cluster

```
${CPDM_OC_LOGIN}
```

- Preview the helm command

  ```
  cpd-cli manage install-components \
  --license_acceptance=true \
  --components=wkc,analyticsengine,dp \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --param-file=/tmp/work/override.yaml \
  --upgrade=true
  ```

  As Upgrade is running, delete this patch: 

  ```bash
  oc patch wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=json --patch'[{"op":"remove","path":"/spec/wdp_policy_service_image"}]'
  ```
- Validate the upgrade

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=ccs,wkc,analyticsengine,dp 
```

#### 2.2.x (Many other services)

#### 2.9 Upgrade the cpdbr service

1.Set the `OADP_OPERATOR_NS` environment variable to the project where the OADP operator is installed:

```
export OADP_OPERATOR_NS=<oadp-operator-project>
```

2.Upgrade the cpdbr-tenant component for the instance.

```
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION} \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION} \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

## Part 3 (Post-upgrade tasks)

### Swapping Models

**(Note: DO NOT PATCH FOR PROD)** Patch the IFM CR with updated models

```
oc patch watsonxaiifm watsonxaiifm-cr \
--namespace=${PROJECT_CPD_INST_OPERANDS} \
--type=merge \
--patch='{"spec":{"install_model_list": ["llama-4-maverick-17b-128e-instruct-fp8","ibm-slate-30m-english-rtrvr","granite-3-2-8b-instruct","mistral-medium-2505","granite-3-3-8b-instruct"]}}'

```

### Update the CPD Route:

Label will be missing from the cpd route post upgrade.

Edit the cpd route to add the label "**expose:external-regional**" to your cpd-route

### Groups Missing:

User groups in access control are missing from UI, but not erased.

Check the zen-metastore-edb postgred DB values:

```
oc rsh zen-metastore-edb-1 -n ${PROJECT_CPD_INST_OPERANDS} 


sh-5.1$ psql -U postgres -d zen

(Note: First 3 command should return the same consistent value. THe last should show only one cpadmin)
zen=# select * from accounts;
zen=# select account_id from user_groups;
zen=# select account_id from platform_users;
zen=# select * from platform_users where username='cpadmin';
```

If there is a difference  "1000" versus "999"

In the same session, back up the table and then update the values to what is shown in accounts (generally 999)

```
CREATE TABLE user_groups_backup AS SELECT * FROM user_groups;

UPDATE user_groups SET account_id = '999';
```

### WxO s3 storage routes

Refernece (Steps below): [https://github.ibm.com/watson-engagement-advisor/wo-cpd-support/blob/main/wxo-support-docs/5.2/5.2.2_storage_s3_route_config_issue_for_nooba.md](https://github.ibm.com/watson-engagement-advisor/wo-cpd-support/blob/main/wxo-support-docs/5.2/5.2.2_storage_s3_route_config_issue_for_nooba.md)

- Identify the s3 route

  ```
  oc get route s3 -n openshift-storage
  ```
- Create rsi patch "wo_custom_s3_route.json" in the "cpd-cli-workspace/olm-utils-workspace/work/rsi" directory

  ```
  [
    { "name": "STORAGE_S3_ROUTE", "value": "<REPLACE WITH YOUR ROUTE>"}
  ]
  ```
- Create and apply the RSI patch for runtime. conversation and archer components.

  ```
  #Create 
  cpd-cli manage create-rsi-patch \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_env_var \
  --patch_name=storage-s3-route-env-override-trm-patch \
  --patch_spec=/tmp/work/rsi/wo_custom_s3_route.json \
  --spec_format=set-env \
  --include_labels=wo.watsonx.ibm.com/component:wo-tools-runtime-manager \
  --state=active

  #Apply
  cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=storage-s3-route-env-override-trm-patch
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  #Create 
  cpd-cli manage create-rsi-patch \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_env_var \
  --patch_name=storage-s3-route-env-override-cc-patch \
  --patch_spec=/tmp/work/rsi/wo_custom_s3_route.json \
  --spec_format=set-env \
  --include_labels=wo.watsonx.ibm.com/component:wo-conversation-controller \
  --state=active

  # Apply
  cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=storage-s3-route-env-override-cc-patch
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  #Create
  cpd-cli manage create-rsi-patch \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --patch_type=rsi_pod_env_var \
  --patch_name=storage-s3-route-env-override-archersvr-patch \
  --patch_spec=/tmp/work/rsi/wo_custom_s3_route.json \
  --spec_format=set-env \
  --include_labels=wo.watsonx.ibm.com/component:wo-archer-server \
  --state=active

  #Apply
  cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --patch_name=storage-s3-route-env-override-archersvr-patch

  ```
- Verify the patch information is present

  ```
   cpd-cli manage get-rsi-patch-info   --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}   --all
  ```

## Summarize and close out the upgrade

1)Prepare for applying the TemporaryPatch if needed as a post-upgrade task.

2)Schedule a wrap-up meeting and review the upgrade procedure and lessons learned from it.

3)Evaluate the outcome of upgrade with pre-defined goals.

---

End of document

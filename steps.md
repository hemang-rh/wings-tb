# RHOAI Installation (imperatively)

## Links

- [Technical Baseline Checklist](https://github.com/redhat-na-ssa/hobbyist-guide-to-rhoai/blob/main/notes/03_CHECKLIST_PROCEDURE.md)

## High Level Steps

- [Fix kubeadmin as an Administrator](#1-fix-kubeadmin-as-an-administrator-for-openshift-ai)
- [Adding adminstrative user](#2-adding-administrative-user)
- [Installing Web Terminal Operator](#3-installing-web-terminal-operator)
- [Installing the Red Hat OpenShift AI Operator using the CLI](#4-installing-the-red-hat-openshift-ai-operator-using-the-cli)
- [Installing and managing Red Hat OpenShift AI components](#5-installing-and-managing-red-hat-openshift-ai-components)
- [Adding a CA bundle](#6-adding-a-ca-bundle)
- [(Optional) Configuring the OpenShift AI Operator logger](#7-optional-configuring-the-openshift-ai-operator-logger)
- Installing KServe dependencies
  - [Installing RHOS ServiceMesh](#81-installing-rhos-servicemesh)
  - [Installing RHOS Serverless](#82-installing-rhos-serverless)

## Step Details

### 1. Fix kubeadmin as an Administrator for Openshift AI

- Create a cluster role binding so that OpenShift AI will recognize `kubeadmin` as a `cluster-admin`

**Commands**

- ```sh
  oc apply -f configs/fix-kubeadmin.yaml
  ```

### 2. Adding administrative user

- Create an htpasswd file to store the user and password information
- Create a secret to represent the htpasswd file
- Define the custom resource for htpasswd
- Apply the resource to the default OAuth configuration to add the identity provider
- As kubeadmin, assign the cluster-admin role to perform administrator level tasks

**Commands**

- ```sh
  htpasswd -c -B -b users.htpasswd admin1 openshift1
  ```

- ```sh
  oc create secret generic htpasswd-secret --from-file=htpasswd=users.htpasswd -n openshift-config
  ```

- ```sh
  oc apply -f configs/htpasswd-cr.yaml
  ```

- ```sh
  oc adm policy add-cluster-role-to-user cluster-admin admin1
  ```

Log in to the cluster as a user from your identity provider, entering the password when prompted
NOTE: You may need to add the parameter `--insecure-skip-tls-verify=true` if your clusters api endpoint does not have a trusted cert.

- ```sh
  oc login --insecure-skip-tls-verify=true -u admin1 -p openshift1
  ```

### 3. Installing Web Terminal Operator

![NOTE] kubeadmin is unable to create web terminals

- Create a subscription object for Web Terminal
- Apply the subscription object

**Commands**

- ```sh
  oc apply -f configs/web-terminal-subscription.yaml
  ```
  ```
  # expected output
  subscription.operators.coreos.com/web-terminal configured
  ```

### 4. Installing the Red Hat OpenShift AI Operator using the CLI

- Create a namespace YAML file
- Apply the namespace object
- Create an OperatorGroup object custom resource (CR) file
- Apply the OperatorGroup object
- Create a Subscription object CR file
- Apply the Subscription object

**Commands**

- ```sh
  oc create -f configs/rhoai-operator-ns.yaml
  ```

- ```sh
  oc create -f configs/rhoai-operator-group.yaml
  ```

- ```sh
  oc create -f configs/rhoai-operator-subscription.yaml
  ```

**Verification**

- Check the installed operators for `rhods-operator.redhat-ods-operator`

  ```sh
  oc get operators
  ```

  ```
  # expected output

  NAME AGE
  devworkspace-operator.openshift-operators 21m
  rhods-operator.redhat-ods-operator 7s
  web-terminal.openshift-operators 22m
  ```

- Check the created projects `redhat-ods-applications|redhat-ods-monitoring|redhat-ods-operator`

  ```sh
  oc get projects | egrep redhat-ods
  ```

  ```
  # expected output
  redhat-ods-applications                                           Active
  redhat-ods-monitoring                                             Active
  redhat-ods-operator                                               Active
  ```

### 5. Installing and managing Red Hat OpenShift AI components

- Create a DataScienceCluster object custom resource (CR) file
- Apply DSC object
- Apply default-dsci object

**Commands**

- ```sh
  oc create -f configs/rhoai-operator-dcs.yaml
  ```

- ```sh
  oc apply -f configs/rhoai-operator-dsci.yaml
  ```

### 6. Adding a CA bundle

- Set environment variables to define base directories for generation of a wildcard certificate and key for the gateways
- Set an environment variable to define the common name used by the ingress controller of your OpenShift cluster
- Set an environment variable to define the domain name used by the ingress controller of your OpenShift cluster

  - ```sh
    export BASE_DIR=/tmp/kserve
    export BASE_CERT_DIR=${BASE_DIR}/certs
    ```

  - ```sh
    export COMMON_NAME=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}' | awk -F'.' '{print $(NF-1)"."$NF}')
    ```

  - ```sh
    export DOMAIN_NAME=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
    ```

- Create the required base directories for the certificate generation, based on the environment variables that you previously set

  - ```sh
    echo $BASE_DIR
    echo $BASE_CERT_DIR
    echo $COMMON_NAME
    echo $DOMAIN_NAME
    mkdir ${BASE_DIR}
    mkdir ${BASE_CERT_DIR}
    ```

- Create the OpenSSL configuration for generation of a wildcard certificate

  - ```sh
    cat <<EOF> ${BASE_DIR}/openssl-san.config
    [ req ]
    distinguished_name = req
    [ san ]
    subjectAltName = DNS:*.${DOMAIN_NAME}
    EOF
    ```

  - ```sh
    cat $BASE_DIR/openssl-san.config
    ```

- Generate a root certificate

  - ```sh
    openssl req -x509 -sha256 -nodes -days 3650 -newkey rsa:2048 \
    -subj "/O=Example Inc./CN=${COMMON_NAME}" \
    -keyout $BASE_DIR/root.key \
    -out $BASE_DIR/root.crt
    ```

  - ```sh
      ls $BASE_DIR/root*
    ```

- Generate a wildcard certificate signed by the root certificate

  - ```sh
    openssl req -x509 -newkey rsa:2048 \
    -sha256 -days 3560 -nodes \
    -subj "/CN=${COMMON_NAME}/O=Example Inc." \
    -extensions san -config ${BASE_DIR}/openssl-san.config \
    -CA $BASE_DIR/root.crt \
    -CAkey $BASE_DIR/root.key \
    -keyout $BASE_DIR/wildcard.key  \
    -out $BASE_DIR/wildcard.crt
    ```

  - ```sh
    openssl x509 -in ${BASE_DIR}/wildcard.crt -text
    ```

- Verify the wildcard certificate

  - ```sh
    openssl verify -CAfile ${BASE_DIR}/root.crt ${BASE_DIR}/wildcard.crt
    ```

    ```
    # expected output
    /tmp/kserve/wildcard.crt: OK
    ```

- Copy the root.crt to paste into default-dcsi yaml
  - ```sh
    cat ${BASE_DIR}/root.crt
    ```
    ```yaml
    # Section in default.dsci yaml that needs to be updated
    spec:
    trustedCABundle:
      customCABundle: |
        -----BEGIN CERTIFICATE-----
        # root.crt value goes here
        -----END CERTIFICATE-----
      managementState: Managed
    ```

**Verification**

- Verify the `odh-trusted-ca-bundle` configmap for your root signed cert in the `odh-ca-bundle.crt:` section

  ```sh
  oc get cm/odh-trusted-ca-bundle -o yaml -n redhat-ods-applications
  ```

- Run the following command to verify that all non-reserved namespaces contain the odh-trusted-ca-bundle ConfigMap

  ```sh
  oc get configmaps --all-namespaces -l app.kubernetes.io/part-of=opendatahub-operator | grep odh-trusted-ca-bundle
  ```

  ```
    # expected output
  istio-system              odh-trusted-ca-bundle   2      10m
  redhat-ods-applications   odh-trusted-ca-bundle   2      10m
  redhat-ods-monitoring     odh-trusted-ca-bundle   2      10m
  redhat-ods-operator       odh-trusted-ca-bundle   2      10m
  rhods-notebooks           odh-trusted-ca-bundle   2      6m55s
  ```

### 7. (Optional) Configuring the OpenShift AI Operator logger

- Configure the log level from the OpenShift CLI by using the following command with the logmode value set to the log level that you want

**Commands**

- ```sh
  oc patch dsci default-dsci -p '{"spec":{"devFlags":{"logmode":"development"}}}' --type=merge
  ```

**Verification**

- Viewing the OpenShift AI Operator log
  ```sh
  oc get pods -l name=rhods-operator -o name -n redhat-ods-operator |  xargs -I {} oc logs -f {} -n redhat-ods-operator
  ```

### 8. Installing KServe dependencies

#### 8.1 Installing RHOS ServiceMesh

    (Optional operators - Kiali, Tempo)
    (Deprecated operators - Jaeger, Elastricsearch)

- Create the required namespace for Red Hat OpenShift Service Mesh
- Define the required subscription for the Red Hat OpenShift Service Mesh Operator
- Create the Service Mesh subscription to install the operator
- Define a ServiceMeshControlPlane object in a YAML file
- Create the servicemesh control plane object

**Commands**

- ```sh
  oc create ns istio-system
  ```

- ```sh
  oc create -f configs/servicemesh-subscription.yaml
  ```

- ```sh
  oc create -f configs/servicemesh-scmp.yaml
  ```

**Verification**

- Verify the pods are running for the service mesh control plane, ingress gateway, and egress gateway

  ```sh
  oc get pods -n istio-system
  ```

#### 8.2 Installing RHOS Serverless

- [ ] Define the Serverless operator namespace, operatorgroup, and subscription
- [ ] Define a ServiceMeshMember object in a YAML file called serverless-smm.yaml
- [ ] Create subscription

**Commands**

- ```sh
  oc create -f configs/serverless-operator.yaml

  # expected output
  namespace/openshift-serverless created
  operatorgroup.operators.coreos.com/serverless-operator created
  subscription.operators.coreos.com/serverless-operator created
  ```

- ```sh
  oc project -n istio-system && oc apply -f configs/serverless-smm.yaml

  # expected output
  servicemeshmember.maistra.io/default created
  ```

#### 3.3 Installing KNative Serving

- [ ] Create knative-serving namespace if it doesn't exist
- [ ] Define a ServiceMeshMember object in a YAML file
- [ ] Create the ServiceMeshMember object in the istio-system namespace
- [ ] Define a KnativeServing object in a YAML file
- [ ] Create the KnativeServing object in the specified knative-serving namespace
- [ ] Verification
  - [ ] Review the default ServiceMeshMemberRoll object in the istio-system namespace and confirm that it includes the knative-serving namespace
  - [ ] Verify creation of the Knative Serving instance

#### 3.4 Creating secure gateways for Knative Serving

- [ ] Set environment variables to define base directories for generation of a wildcard certificate and key for the gateways.
- [ ] Set an environment variable to define the common name used by the ingress controller of your OpenShift cluster
- [ ] Create the required base directories for the certificate generation, based on the environment variables that you previously set
- [ ] Create the OpenSSL configuration for generation of a wildcard certificate
- [ ] Generate a root certificate
- [ ] Generate a wildcard certificate signed by the root certificate
- [ ] Verify the wildcard certificate
- [ ] Export the wildcard key and certificate that were created by the script to new environment variables
- [ ] Create a TLS secret in the istio-system namespace using the environment variables that you set for the wildcard certificate and key
- [ ] Create a serverless-gateways.yaml YAML file
- [ ] Apply the serverless-gateways.yaml file to create the defined resources
- [ ] Review the gateways that you created

#### 3.5 Installing Kserve

- [ ] Set serviceMesh component as "managementState: Unmanaged" (inside default-dsci)

```

spec:
serviceMesh:
managementState: Unmanaged

```

- [ ] Set kserve component as "managementState: Managed" (inside default-dsc)
- [ ] Set serving component within kserve component as "managementState: Unmanaged" (inside default-dsc)

```

spec:
components:
kserve:
managementState: Managed
serving:
managementState: Unmanaged

```

#### 3.6 Manually adding an authorization provider

- [ ] Create subscription for the Authorino Operator
- [ ] Install the Authorino operator
- [ ] Create a namespace to install the Authorino instance
- [ ] Enroll the new namespace for the Authorino instance in your existing OpenShift Service Mesh instance
- [ ] Create the ServiceMeshMember resource on your cluster
- [ ] Configure an Authorino instance, create a new YAML file as shown
- [ ] Create the Authorino resource on your cluster
- [ ] Patch the Authorino deployment to inject an Istio sidecar, which makes the Authorino instance part of your OpenShift Service Mesh instance
- [ ] Check the pods (and containers) that are running in the namespace that you created for the Authorino instance, as shown in the following example

#### 3.7 Configuring an OpenShift Service Mesh instance to use Authorino

- [ ] Create a new YAML file to patch the ServiceMesh Control Plane
- [ ] Use the oc patch command to apply the YAML file to your OpenShift Service Mesh instance
- [ ] Verification

- [ ] Inspect the ConfigMap object for your OpenShift Service Mesh instance (should look similar to below)

```

defaultConfig:
discoveryAddress: istiod-data-science-smcp.istio-system.svc:15012
proxyMetadata:
ISTIO_META_DNS_AUTO_ALLOCATE: "true"
ISTIO_META_DNS_CAPTURE: "true"
PROXY_XDS_VIA_AGENT: "true"
terminationDrainDuration: 35s
tracing: {}
dnsRefreshRate: 300s
enablePrometheusMerge: true
extensionProviders:

- envoyExtAuthzGrpc:
  port: 50051
  service: authorino-authorino-authorization.opendatahub-auth-provider.svc.cluster.local
  name: opendatahub-auth-provider
  ingressControllerMode: "OFF"
  rootNamespace: istio-system
  trustDomain: null%

```

### 3.8 Configuring authorization for KServe

- [ ] Create a new YAML file for
- [ ] Create the AuthorizationPolicy resource in the namespace for your OpenShift Service Mesh instance
- [ ] Create another new YAML file with the following contents:
- [ ] Create the EnvoyFilter resource in the namespace for your OpenShift Service Mesh instance
- [ ] Check that the AuthorizationPolicy resource was successfully created
- [ ] Check that the EnvoyFilter resource was successfully created

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

```

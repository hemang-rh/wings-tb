# RHOAI Installation (imperatively)

## Links

- [Technical Baseline Checklist](https://github.com/redhat-na-ssa/hobbyist-guide-to-rhoai/blob/main/notes/03_CHECKLIST_PROCEDURE.md)

## High Level Steps

- [Fix kubeadmin as an administrator](#1-fix-kubeadmin-as-an-administrator-for-openshift-ai)
- [Adding adminstrative user](#2-adding-administrative-user)
- [Installing Web Terminal operator](#3-installing-web-terminal-operator)
- [Installing the Red Hat OpenShift AI operator using the CLI](#4-installing-the-red-hat-openshift-ai-operator-using-the-cli)
- [Installing and managing Red Hat OpenShift AI components](#5-installing-and-managing-red-hat-openshift-ai-components)
- [Adding a CA bundle](#6-adding-a-ca-bundle)
- [(Optional) Configuring the OpenShift AI operator logger](#7-optional-configuring-the-openshift-ai-operator-logger)
- [Installing RHOS ServiceMesh](#8-installing-rhos-servicemesh)
- [Installing RHOS Serverless](#9-installing-rhos-serverless)
- [Feature Tracker error fix](#10-feature-tracker-error-fix)
- [Creating KNative serving instance](#11-creating-a-knative-serving-instance)
- [Creating secure gateways for KNative serving](#12-creating-secure-gateways-for-knative-serving)
- [Manually adding Authorization provider](#13-manually-adding-authorizaiton-provider)

## Step Details

### 1. Fix kubeadmin as an Administrator for Openshift AI

> `kubeadmin` user is an automatically generated temporary user.  
> Best practice is to create a new user using an identity provider and elevate the privileges of that user to cluster-admin.  
> Once such user is created, the default kubeadmin user should be removed [More Info](https://access.redhat.com/solutions/5309141)

- Create a cluster role binding so that OpenShift AI will recognize `kubeadmin` as a `cluster-admin`

  - ```sh
    oc apply -f configs/fix-kubeadmin.yaml
    ```

### 2. Adding administrative user

> For this process we are using HTpasswd typical for PoC.  
>  You can configure the following types of identity providers htpasswd, keystone, LDAP, basic-authentication, request-header, GitHub, GitLab, Google, OpenID Connect.

- Create an htpasswd file to store the user and password information
- Create a secret to represent the htpasswd file
- Define the custom resource for htpasswd
- Apply the resource to the default OAuth configuration to add the identity provider
- As kubeadmin, assign the cluster-admin role to perform administrator level tasks

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

- Log in to the cluster as a user from your identity provider, entering the password when prompted.  
  NOTE: You may need to add the parameter `--insecure-skip-tls-verify=true` if your clusters api endpoint does not have a trusted cert.
  - ```sh
    oc login --insecure-skip-tls-verify=true -u admin1 -p openshift1
    ```

### 3. Installing Web Terminal Operator

> This provides a Web Terminal in the same browser as the OCP Web Console to minimize context switching between the browser and local client. [More Info](https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/web_console/web-terminal)

> ![NOTE] kubeadmin is unable to create web terminals [More Info](https://github.com/redhat-developer/web-terminal-operator/issues/162)

- Create a subscription object for Web Terminal
- Apply the subscription object

  - ```sh
    oc apply -f configs/web-terminal-subscription.yaml
    ```
  - ```sh
    # expected output
    subscription.operators.coreos.com/web-terminal configured
    ```

### 4. Installing the Red Hat OpenShift AI Operator using the CLI

> [More Info](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.10/html/installing_and_uninstalling_openshift_ai_self-managed/installing-and-deploying-openshift-ai_install#installing-the-openshift-data-science-operator_operator-install)

> Understanding update channels. We are using fast channel as this gives customers access to the latest product features [More Info](https://access.redhat.com/support/policy/updates/rhoai-sm/lifecycle)

- Create a namespace YAML file
- Apply the namespace object
- Create an OperatorGroup object custom resource (CR) file
- Apply the OperatorGroup object
- Create a Subscription object CR file
- Apply the Subscription object

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

  - ```sh
    oc get operators
    ```

    ```sh
    # expected output
    NAME AGE
    devworkspace-operator.openshift-operators 21m
    rhods-operator.redhat-ods-operator 7s
    web-terminal.openshift-operators 22m
    ```

- Check the created projects `redhat-ods-applications|redhat-ods-monitoring|redhat-ods-operator`

  - ```sh
    oc get projects | egrep redhat-ods
    ```

    ```
    # expected output

    redhat-ods-applications Active
    redhat-ods-monitoring Active
    redhat-ods-operator Active
    ```

### 5. Installing and managing Red Hat OpenShift AI components

- Create a DataScienceCluster object custom resource (CR) file
- Apply DSC object
- Apply default-dsci object

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

  - ```sh
    oc get cm/odh-trusted-ca-bundle -o yaml -n redhat-ods-applications
    ```

- Run the following command to verify that all non-reserved namespaces contain the odh-trusted-ca-bundle ConfigMap

  - ```sh
    oc get configmaps --all-namespaces -l app.kubernetes.io/part-of=opendatahub-operator | grep odh-trusted-ca-bundle
    ```

    ```
    # expected output

    istio-system odh-trusted-ca-bundle 2 10m
    redhat-ods-applications odh-trusted-ca-bundle 2 10m
    redhat-ods-monitoring odh-trusted-ca-bundle 2 10m
    redhat-ods-operator odh-trusted-ca-bundle 2 10m
    rhods-notebooks odh-trusted-ca-bundle 2 6m55s

    ```

### 7. (Optional) Configuring the OpenShift AI Operator logger

- Configure the log level from the OpenShift CLI by using the following command with the logmode value set to the log level that you want

  - ```sh
    oc patch dsci default-dsci -p '{"spec":{"devFlags":{"logmode":"development"}}}' --type=merge
    ```

**Verification**

- Viewing the OpenShift AI Operator log
  - ```sh
    oc get pods -l name=rhods-operator -o name -n redhat-ods-operator |  xargs -I {} oc logs -f {} -n redhat-ods-operator
    ```

### 8. Installing RHOS ServiceMesh

    (Optional operators - Kiali, Tempo)
    (Deprecated operators - Jaeger, Elastricsearch)

- Create the required namespace for Red Hat OpenShift Service Mesh
- Define the required subscription for the Red Hat OpenShift Service Mesh Operator
- Create the Service Mesh subscription to install the operator
- Define a ServiceMeshControlPlane object in a YAML file
- Create the servicemesh control plane object

  - ```sh
    oc create ns istio-system
    ```

    ```sh
    # expected output
    namespaces "istio-system" already exists
    ```

  - ```sh
    oc create -f configs/servicemesh-subscription.yaml
    ```

    ```sh
    # expected output
    subscription.operators.coreos.com/servicemeshoperator created

    Note: Wait for Service Mesh opertor to be installed before creating servicemesh control plane object. If you try to create before that you will get below error

    # error
    error: resource mapping not found for name: "minimal" namespace: "istio-system" from "configs/servicemesh-scmp.yaml": no matches for kind "ServiceMeshControlPlane" in version "maistra.io/v2" ensure CRDs are installed first
    ```

  - ```sh
    oc create -f configs/servicemesh-scmp.yaml
    ```

    ```sh
    # expected output
    servicemeshcontrolplane.maistra.io/minimal created
    ```

**Verification**

- Verify the pods are running for the service mesh control plane, ingress gateway, and egress gateway

  - ```sh
    oc get pods -n istio-system
    ```

    ```sh
    # expected output
    istio-egressgateway-f9b5cf49c-c7fst    1/1     Running   0          59s
    istio-ingressgateway-c69849d49-fjswg   1/1     Running   0          59s
    istiod-minimal-5c68bf675d-whrns        1/1     Running   0          68s
    ```

### 9. Installing RHOS Serverless

- Define the Serverless operator namespace, operatorgroup, and subscription
- Define a ServiceMeshMember object in a YAML file called serverless-smm.yaml
- Create subscription

- ```sh
  oc create -f configs/serverless-operator.yaml
  ```

  ```sh
  # expected output

  namespace/openshift-serverless created
  operatorgroup.operators.coreos.com/serverless-operator created
  subscription.operators.coreos.com/serverless-operator created

  ```

- ```sh
  oc project -n istio-system && oc apply -f configs/serverless-smm.yaml
  ```

  ```
  # expected output
  servicemeshmember.maistra.io/default created
  ```

### 10. Feature Tracker Error Fix

There are two objects that are in an error state after installation at this point.

- redhat-ods-applications-mesh-metrics-collection
- redhat-ods-applications-mesh-control-plane-creation

  - ```sh
    # get the mutatingwebhook
    oc get MutatingWebhookConfiguration -A | grep -i maistra

    # delete the mutatingwebhook
    oc delete MutatingWebhookConfiguration/openshift-operators.servicemesh-resources.maistra.io -A
    ```

  - ```sh
    # get the validatingwebhook
    oc get ValidatingWebhookConfiguration -A | grep -i maistra

    # delete the validatingwebhook
    oc delete ValidatingWebhookConfiguration/openshift-operators.servicemesh-resources.maistra.io -A
    ```

  - ```sh
    # delete the FeatureTracker
    oc delete FeatureTracker/redhat-ods-applications-mesh-control-plane-creation -A
    oc delete FeatureTracker/redhat-ods-applications-mesh-metrics-collection -A
    ```

### 11. Creating a KNative Serving Instance

[Section 3.3.1.2 source](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.10/html/serving_models/serving-large-models_serving-large-models#creating-a-knative-serving-instance_serving-large-models)

- Define a KnativeServing object in a YAML file
- Apply the KnativeServing object in the specified knative-serving namespace

  - ```sh
    oc create -f configs/serverless-istio.yaml
    ```
    ```
    # expected output
    knativeserving.operator.knative.dev/knative-serving created
    ```

- Review the default ServiceMeshMemberRoll object in the istio-system namespace and confirm that it includes the knative-serving namespace

  - ```sh
    oc describe smmr default -n istio-system
    ```
    ```
    # expected output
    ...
    Member Statuses:
        Conditions:
        Last Transition Time:  2024-07-25T22:15:59Z
        Status:                True
        Type:                  Reconciled
        Namespace:               knative-serving
    Members:
        knative-serving
    ...
    ```
  - ```sh
    oc get smmr default -n istio-system -o jsonpath='{.status.memberStatuses}'
    ```
    ```
    # expected output TODO
    [{"conditions":[{"lastTransitionTime":"2024-07-16T18:09:10Z","status":"Unknown","type":"Reconciled"}],"namespace":"knative-serving"}]
    ```

- Verify creation of the Knative Serving instance
  - ```sh
    oc get pods -n knative-serving
    ```
    ```
    # expected output
    activator-5cf876c6cf-jtvs2                                    0/1     Running     0             91s
    activator-5cf876c6cf-ntf79                                    0/1     Running     0             76s
    autoscaler-84655b4df5-w9lmc                                   1/1     Running     0             91s
    autoscaler-84655b4df5-zznlw                                   1/1     Running     0             91s
    autoscaler-hpa-986bb8687-llms8                                1/1     Running     0             90s
    autoscaler-hpa-986bb8687-qtgln                                1/1     Running     0             90s
    controller-84cb7b64bc-9654q                                   1/1     Running     0             89s
    controller-84cb7b64bc-bdhps                                   1/1     Running     0             83s
    net-istio-controller-6498db6ccb-4ddvd                         0/1     Running     2 (24s ago)   89s
    net-istio-controller-6498db6ccb-f66mv                         0/1     Running     2 (24s ago)   89s
    net-istio-webhook-79cbc7c4d4-r6gln                            1/1     Running     0             89s
    net-istio-webhook-79cbc7c4d4-snd7k                            1/1     Running     0             89s
    storage-version-migration-serving-serving-1.12-1.33.0-6v9ll   0/1     Completed   0             89s
    webhook-6bb9cd8c97-46lz4                                      1/1     Running     0             90s
    webhook-6bb9cd8c97-cxm2n                                      1/1     Running     0             75s
    ```

### 12. Creating secure gateways for Knative Serving

> [More Info](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.10/html/serving_models/serving-large-models_serving-large-models#creating-secure-gateways-for-knative-serving_serving-large-models)

> Why? To secure traffic between your Knative Serving instance and the service mesh, you must create secure gateways for your Knative Serving instance.  
> The initial steps to generate a root signed certificate were completed previous

- Verify the wildcard certificate
  - ```sh
    openssl verify -CAfile ${BASE_DIR}/root.crt ${BASE_DIR}/wildcard.crt
    ```
    ```
    # expected output
    /tmp/kserve/wildcard.crt: OK
    ```
- Export the wildcard key and certificate that were created by the script to new environment variables
  - ```sh
    export TARGET_CUSTOM_CERT=${BASE_DIR}/wildcard.crt
    export TARGET_CUSTOM_KEY=${BASE_DIR}/wildcard.key
    ```
- Create a TLS secret in the istio-system namespace using the environment variables that you set for the wildcard certificate and key
  - ```sh
    oc create secret tls wildcard-certs --cert=${TARGET_CUSTOM_CERT} --key=${TARGET_CUSTOM_KEY} -n istio-system
    ```
    ```
    # expected output
    secret/wildcard-certs created
    ```

> Define a service in the istio-system namespace for the Knative local gateway. Defines an ingress gateway in the knative-serving namespace. The gateway uses the TLS secret you created earlier in this procedure. The ingress gateway handles external traffic to Knative. Defines a local gateway for Knative in the knative-serving namespace.

- Create a serverless-gateway.yaml YAML file
- Apply the serverless-gateways.yaml file to create the defined resources
  - ```sh
    oc apply -f configs/serverless-gateway.yaml
    ```
    ```
    # expected output
    service/knative-local-gateway unchanged
    gateway.networking.istio.io/knative-ingress-gateway created
    gateway.networking.istio.io/knative-local-gateway created
    ```
- Review the gateways that you created
  - ```sh
    oc get gateway --all-namespaces
    ```
    ```
    # expected output
    NAMESPACE         NAME                      AGE
    knative-serving   knative-ingress-gateway   2m
    knative-serving   knative-local-gateway     2m
    ```

### 13. Manually adding Authorizaiton provider

> Why? Adding an authorization provider allows you to enable token authorization for models that you deploy on the platform, which ensures that only authorized parties can make inference requests to the models. [More Info](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.10/html/serving_models/serving-large-models_serving-large-models#manually-adding-an-authorization-provider_serving-large-models)

- Create subscription for the Authorino Operator
- Apply the Authorino operator

  - ```sh
    oc create -f configs/authorino-subscription.yaml
    ```
    ```
    # expected output
    subscription.operators.coreos.com/authorino-operator created
    ```

- Create a namespace to install the Authorino instance

  - ```sh
    oc create ns redhat-ods-applications-auth-provider
    ```
    ```
    # expected output
    namespace/redhat-ods-applications-auth-provider created
    ```

- Enroll the new namespace for the Authorino instance in your existing OpenShift Service Mesh instance
- Create the ServiceMeshMember resource on your cluster

  - ```sh
    oc create -f configs/authorino-smm.yaml
    ```
    ```
    # expected output
    servicemeshmember.maistra.io/default created
    ```

- Configure an Authorino instance,
- Create the Authorino resource on your cluster

  - ```sh
    oc create -f configs/authorino-instance.yaml
    ```
    ```
    # expected output
    authorino.operator.authorino.kuadrant.io/authorino created
    ```

- Patch the Authorino deployment to inject an Istio sidecar, which makes the Authorino instance part of your OpenShift Service Mesh instance

  - ```sh
    oc patch deployment authorino -n redhat-ods-applications-auth-provider -p '{"spec": {"template":{"metadata":{"labels":{"sidecar.istio.io/inject":"true"}}}} }'
    ```
    ```
    # expected output
    deployment.apps/authorino patched
    ```

- Check the pods (and containers) that are running in the namespace that you created for the Authorino instance, as shown in the following example

  - ```sh
    oc get pods -n redhat-ods-applications-auth-provider -o="custom-columns=NAME:.metadata.name,STATUS:.status.phase,CONTAINERS:.spec.containers[*].name"
    ```
    ```
    # expected output
    NAME                         STATUS    CONTAINERS
    authorino-75585d99bd-vh65n   Running   authorino,istio-proxy
    ```

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

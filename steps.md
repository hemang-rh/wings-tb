# RHOAI Installation (imperatively)

## Links

- [Technical Baseline Checklist](https://github.com/redhat-na-ssa/hobbyist-guide-to-rhoai/blob/main/notes/03_CHECKLIST_PROCEDURE.md)

## High Level Steps

- [Fix kubeadmin as an administrator](#1-fix-kubeadmin-as-an-administrator-for-openshift-ai)
- [Adding administrative user](#2-adding-administrative-user)
- [(Optional) Installing Web Terminal operator](#3-installing-web-terminal-operator)
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
- [Configuring an OpenShift Service Mesh instance to use Authorino](#14-configuring-an-openshift-service-mesh-instance-to-use-authorino)
- [Configuring authorization for KServe](#15-configuring-authorization-for-kserve)
- [Adding GPU node to an existing OCP cluster](#16-adding-a-gpu-node-to-an-existing-openshift-container-platform-cluster)
- [Deploying the Node Feature Discovery operator](#17-deploying-the-node-feature-discovery-operator)
- [Installing the NVidia GPU operator](#18-installing-the-nvidia-gpu-operator)
- [(Optional) Running a sample GPU application](#19-optional-running-a-sample-gpu-application)
- [Enable GPU monitoring dashboard](#20-enable-gpu-monitoring-dashboard)
- [Installing the NVIDIA GPU administration dashboard](#21-installing-the-nvidia-gpu-administration-dashboard)
- [Configuring GPUs with time slicing](#22-configuring-gpus-with-time-slicing)
- [Configure taints and tolerations](#23-configure-taints-and-tolerations)

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
    htpasswd -c -B -b scratch/users.htpasswd admin1 openshift1
    ```

  - ```sh
    oc create secret generic htpasswd-secret --from-file=htpasswd=scratch/users.htpasswd -n openshift-config
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

### 3. (Optional) Installing Web Terminal Operator

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

> Why? Adding an authorization provider allows you to enable token authorization for models that you deploy on the platform, which ensures that only authorized parties can make inference requests to the models.  
> [More Info](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.10/html/serving_models/serving-large-models_serving-large-models#manually-adding-an-authorization-provider_serving-large-models)

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

### 14. Configuring an OpenShift Service Mesh instance to use Authorino

> Why? you must configure your OpenShift Service Mesh instance to use Authorino as an authorization provider  
> [More Info](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.10/html/serving_models/serving-large-models_serving-large-models#configuring-service-mesh-instance-to-use-authorino_serving-large-models)

- Create a new YAML file with the following contents servicemesh-smcp-patch.yaml
- Use the oc patch command to apply the YAML file to your OpenShift Service Mesh instance

  - ```sh
    oc patch smcp minimal --type merge -n istio-system --patch-file configs/files/servicemesh-smcp-patch.yaml
    ```
    ```
    # expected output
    servicemeshcontrolplane.maistra.io/minimal patched
    ```

- Inspect the ConfigMap object for your OpenShift Service Mesh instance
  - ```sh
    oc get configmap istio-minimal -n istio-system --output=jsonpath={.data.mesh}
    ```
    ```
    # expected output
    defaultConfig:
    discoveryAddress: istiod-minimal.istio-system.svc:15012
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
        service: authorino-authorino-authorization.redhat-ods-applicatiions-auth-provider.svc.cluster.local
    name: redhat-ods-applications-auth-provider
    ingressControllerMode: "OFF"
    rootNamespace: istio-system
    trustDomain: null
    ```
- Confirm that you see output that the Authorino instance has been successfully added as an extension provider

### 15. Configuring authorization for KServe

> Why? you must create a global AuthorizationPolicy resource that is applied to the KServe predictor pods that are created when you deploy a model. In addition, to account for the multiple network hops that occur when you make an inference request to a model, you must create an EnvoyFilter resource that continually resets the HTTP host header to the one initially included in the inference request.  
> [More Info](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.10/html/serving_models/serving-large-models_serving-large-models#configuring-authorization-for-kserve_serving-large-models)

- Create a new YAML file
- Create the AuthorizationPolicy resource in the namespace for your OpenShift Service Mesh instance

  - ```sh
    oc create -n istio-system -f configs/servicemesh-authorization-policy.yaml
    ```
    ```
    # expected output
    authorizationpolicy.security.istio.io/kserve-predictor created
    ```

- Create another new YAML file for EnvoyFilter: The EnvoyFilter resource continually resets the HTTP host header to the one initially included in any inference request.
- Create the EnvoyFilter resource in the namespace for your OpenShift Service Mesh instance

  - ```sh
    oc create -n istio-system -f configs/servicemesh-envoyfilter.yaml
    ```
    ```
    # expected output
    envoyfilter.networking.istio.io/activator-host-header created
    ```

- Check that the AuthorizationPolicy resource was successfully created.

  - ```sh
    oc get authorizationpolicies -n istio-system
    ```
    ```
    # expected output
    NAME               AGE
    kserve-predictor   62s
    ```

- Check that the EnvoyFilter resource was successfully created.
  - ```sh
    oc get envoyfilter -n istio-system
    ```
    ```
    # example output
    NAME                                AGE
    activator-host-header               101s
    metadata-exchange-1.6-minimal       56m
    tcp-metadata-exchange-1.6-minimal   56m
    ```

### Enabling GPU support in OpenShift AI

> [More Info](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.10/html/installing_and_uninstalling_openshift_ai_self-managed/enabling-gpu-support_install)

---

### 16. Adding a GPU node to an existing OpenShift Container Platform cluster

> [More Info](https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/machine_management/managing-compute-machines-with-the-machine-api#nvidia-gpu-aws-adding-a-gpu-node_creating-machineset-aws)

- View existing nodes

  - ```sh
    oc get nodes
    ```
    ```
    # expected output
    NAME                                        STATUS   ROLES                         AGE     VERSION
    ip-10-x-xx-xxx.us-east-x.compute.internal   Ready    control-plane,master,worker   5h11m   v1.28.10+a2c84a5
    ```

- View the machine sets that exist in the openshift-machine-api namespace

  - ```sh
    oc get machinesets -n openshift-machine-api
    ```
    ```
    # expected output
    NAME                                    DESIRED   CURRENT   READY   AVAILABLE   AGE
    cluster-xxxxx-xxxxx-worker-us-east-xc   0         0                             5h13m
    ```

- View the machines that exist in the openshift-machine-api namespace

  - ```sh
    oc get machines -n openshift-machine-api | egrep worker
    ```

- Make a copy of one of the existing compute MachineSet definitions and output the result to a YAML file
  - ```sh
    oc get machineset -n openshift-machine-api
    ```
  - ```sh
    oc get machineset <machineset-name> -n openshift-machine-api -o yaml > scratch/machineset.yaml
    ```

> Update the following fields:

- > .spec.replicas from 0 to 2
- > .metadata.name to a name containing gpu.
- > .spec.selector.matchLabels["machine.openshift.io/cluster-api-machineset"] to match the new .metadata.name.
- > .spec.template.metadata.labels["machine.openshift.io/cluster-api-machineset"] to match the new .metadata.name.
- > .spec.template.spec.providerSpec.value.instanceType to g4dn.4xlarge.

> Remove the following fields

- > uid
- > generation

- Apply the configuration to create the gpu machine

  - ```sh
    oc apply -f scratch/machineset.yaml
    ```
    ```
    # expected output
    machineset.machine.openshift.io/cluster-xxxx-xxxx-worker-us-east-gpu created
    ```

- Verify the gpu machineset you created is running

  - ```sh
    oc -n openshift-machine-api get machinesets | grep gpu
    ```
    ```
    # expected output
    cluster-xxxxx-xxxxx-worker-us-east-xc-gpu   2         2         2       2           6m37s
    ```

- View the Machine object that the machine set created
  - ```sh
    oc -n openshift-machine-api get machines | grep gpu
    ```
    ```
    # expected output
    cluster-xxxxx-xxxxx-worker-us-east-xc-gpu-29whc   Running   g4dn.4xlarge   us-east-2   us-east-2c   7m59s
    cluster-xxxxx-xxxxx-worker-us-east-xc-gpu-nr59d   Running   g4dn.4xlarge   us-east-2   us-east-2c   7m59s
    ```

### 17. Deploying the Node Feature Discovery operator

> [More Info](https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/machine_management/managing-compute-machines-with-the-machine-api#nvidia-gpu-aws-deploying-the-node-feature-discovery-operator_creating-machineset-aws)

- List the available operators for installation searching for Node Feature Discovery (NFD)

  - ```sh
    oc get packagemanifests -n openshift-marketplace | grep nfd
    ```
    ```
    # expected output
    openshift-nfd-operator                             Community Operators   8h
    nfd                                                Red Hat Operators     8h
    ```

- Create and apply the Namespace object

  - ```sh
    oc apply -f configs/nfd-operator-ns.yaml
    ```
    ```
    # expected output
    # expected output
    namespace/openshift-nfd created
    ```

- Create and apply the OperatorGroup object

  - ```sh
    oc apply -f configs/nfd-operator-group.yaml
    ```
    ```
    # expected output
    operatorgroup.operators.coreos.com/nfd created
    ```

- Create a Subscription object YAML file to subscribe a namespace to an Operator
- Apply the Subscription object

  - ```sh
    oc apply -f configs/nfd-operator-sub.yaml
    ```
    ```
    # expected output
    subscription.operators.coreos.com/nfd created
    ```

- Verify the operator is installed and running

  - ```sh
    oc get pods -n openshift-nfd
    ```
    ```
    # expected output
    NAME                                      READY   STATUS    RESTARTS   AGE
    nfd-controller-manager-78758c57f7-7xfh4   2/2     Running   0          48s
    ```

- Create an NodeFeatureDiscovery instance via the CLI or UI (recommended)
- Create the nfd instance object
  - ```sh
    oc apply -f configs/nfd-instance.yaml
    ```
    ```
    # expected output
    nodefeaturediscovery.nfd.openshift.io/nfd-instance created
    ```

> ![IMPORTANT] The NFD Operator uses vendor PCI IDs to identify hardware in a node. NVIDIA uses the PCI ID 10de.

- Verify the NFD pods are Running on the cluster nodes polling for devices

  - ```sh
    oc get pods -n openshift-nfd
    ```
    ```
    # expected output
    NAME                                      READY   STATUS    RESTARTS   AGE
    nfd-controller-manager-78758c57f7-7xfh4   2/2     Running   0          99s
    nfd-master-74db665cb6-vht4l               1/1     Running   0          25s
    nfd-worker-8zkpz                          1/1     Running   0          25s
    nfd-worker-d7wgh                          1/1     Running   0          25s
    nfd-worker-l6sqx                          1/1     Running   0          25s
    ```

- Verify the NVIDIA GPU is discovered
  - ```sh
    oc get nodes
    ```
  - ```sh
    oc describe node <NODE_NAME> | egrep 'Roles|pci'
    ```
    ```
    # expected output
    Roles:              worker
                        feature.node.kubernetes.io/pci-10de.present=true
                        feature.node.kubernetes.io/pci-1d0f.present=true
    ```

> Verify the NVIDIA GPU is discovered 10de appears in the node feature list for the GPU-enabled node. This mean the NFD Operator correctly identified the node from the GPU-enabled MachineSet.

### 18. Installing the NVIDIA GPU Operator

> [More Info](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/install-gpu-ocp.html#installing-the-nvidia-gpu-operator-using-the-cli)

- List the available operators for installation searching for Node Feature Discovery (NFD)

  - ```sh
    oc get packagemanifests -n openshift-marketplace | grep gpu
    ```

    ```
    # expected output
    amd-gpu-operator                                   Community Operators   8h
    gpu-operator-certified                             Certified Operators   8h
    ```

- Create a Namespace custom resource (CR) that defines the nvidia-gpu-operator namespace
- Apply the Namespace object YAML file

  - ```sh
    oc apply -f configs/nvidia-gpu-operator-ns.yaml
    ```

    ```sh
    # expected output
    namespace/nvidia-gpu-operator created
    ```

- Create an OperatorGroup CR
- Apply the OperatorGroup YAML file

  - ```sh
    oc apply -f configs/nvidia-gpu-operator-group.yaml
    ```

    ```sh
    # expected output
    operatorgroup.operators.coreos.com/nvidia-gpu-operator-group created
    ```

- Run the following command to get the channel value

  - ```sh
    oc get packagemanifest gpu-operator-certified -n openshift-marketplace -o jsonpath='{.status.defaultChannel}'
    ```

    ```sh
    # expected output
    v24.3
    ```

- Run the following commands to get the startingCSV

  - ```sh
    CHANNEL=<channel-from-previous-ouput>

    oc get packagemanifests/gpu-operator-certified -n openshift-marketplace -ojson | jq -r '.status.channels[] | select(.name == "'$CHANNEL'") | .currentCSV'
    ```

    ```sh
    # expected output
    gpu-operator-certified.v24.3.0
    ```

- Create the following Subscription CR and save the YAML
- Update the `channel` and `startingCSV` fields with the information returned
- Apply the Subscription CR

  - ```sh
    oc apply -f configs/nvidia-gpu-operator-subscription.yaml
    ```

    ```sh
    # expected output
    subscription.operators.coreos.com/gpu-operator-certified created
    ```

- Verify an install plan has been created

  - ```sh
    oc get installplan -n nvidia-gpu-operator
    ```

    ```sh
    # expected output
    NAME            CSV                              APPROVAL    APPROVED
    install-q9rnm   gpu-operator-certified.v24.3.0   Automatic   true
    ```

**(Optional) Approve the install plan if not `Automatic`**

- ```sh
  INSTALL_PLAN=$(oc get installplan -n nvidia-gpu-operator -oname)
  ```

- Create the cluster policy

  - ```sh
    oc get csv -n nvidia-gpu-operator gpu-operator-certified.v24.3.0 -o jsonpath='{.metadata.annotations.alm-examples}' | jq '.[0]' > scratch/nvidia-gpu-clusterpolicy.json
    ```

- Apply the clusterpolicy

  - ```sh
    oc apply -f scratch/nvidia-gpu-clusterpolicy.json
    ```

    ```sh
    # expected output
    clusterpolicy.nvidia.com/gpu-cluster-policy created
    ```

> At this point, the GPU Operator proceeds and installs all the required components to set up the NVIDIA GPUs in the OpenShift 4 cluster. Wait at least 10-20 minutes before digging deeper into any form of troubleshooting because this may take a period of time to finish.

- Verify the successful installation of the NVIDIA GPU Operator

  - ```sh
    oc get pods,daemonset -n nvidia-gpu-operator
    ```

    ```sh
    # expected output
    NAME                                                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                                                                                         AGE
    daemonset.apps/gpu-feature-discovery                           0         0         0       0            0           nvidia.com/gpu.deploy.gpu-feature-discovery=true                                                                      22s
    daemonset.apps/nvidia-container-toolkit-daemonset              0         0         0       0            0           nvidia.com/gpu.deploy.container-toolkit=true                                                                          22s
    daemonset.apps/nvidia-dcgm                                     0         0         0       0            0           nvidia.com/gpu.deploy.dcgm=true                                                                                       22s
    daemonset.apps/nvidia-dcgm-exporter                            0         0         0       0            0           nvidia.com/gpu.deploy.dcgm-exporter=true                                                                              22s
    daemonset.apps/nvidia-device-plugin-daemonset                  0         0         0       0            0           nvidia.com/gpu.deploy.device-plugin=true                                                                              22s
    daemonset.apps/nvidia-device-plugin-mps-control-daemon         0         0         0       0            0           nvidia.com/gpu.deploy.device-plugin=true,nvidia.com/mps.capable=true                                                  22s
    daemonset.apps/nvidia-driver-daemonset-415.92.202406251950-0   2         2         0       2            0           feature.node.kubernetes.io/system-os_release.OSTREE_VERSION=415.92.202406251950-0,nvidia.com/gpu.deploy.driver=true   22s
    daemonset.apps/nvidia-mig-manager                              0         0         0       0            0           nvidia.com/gpu.deploy.mig-manager=true                                                                                22s
    daemonset.apps/nvidia-node-status-exporter                     2         2         2       2            2           nvidia.com/gpu.deploy.node-status-exporter=true                                                                       22s
    daemonset.apps/nvidia-operator-validator                       0         0         0       0            0           nvidia.com/gpu.deploy.operator-validator=true                                                                         22s
    ```

**(Optional) When the NVIDIA operator completes labeling the nodes, you can add a label to the GPU node Role as `gpu, worker` for readability (cosmetic)**

- ```sh
  oc label node -l nvidia.com/gpu.machine node-role.kubernetes.io/gpu=''
  ```

- ```sh
  oc get nodes
  ```

  ```
  # expected output
  NAME                                        STATUS   ROLES                         AGE   VERSION
  ip-10-0-xx-xxx.us-east-2.compute.internal   Ready    gpu,worker                    19h   v1.28.10+a2c84a5
  ip-10-0-xx-xxx.us-east-2.compute.internal   Ready    gpu,worker                    19h   v1.28.10+a2c84a5
  ...
  ```

- In order to apply this label to new machines/nodes:

  - ```sh
    MACHINE_SET_TYPE=$(oc -n openshift-machine-api get machinesets.machine.openshift.io -o name | grep gpu | head -n1)

    oc -n openshift-machine-api \
      patch "${MACHINE_SET_TYPE}" \
      --type=merge --patch '{"spec":{"template":{"spec":{"metadata":{"labels":{"node-role.kubernetes.io/gpu":""}}}}}}'
    ```

    ```sh
    # expected output
    machineset.machine.openshift.io/cluster-xxxxx-xxxxx-worker-us-east-xc-gpu patched
    ```

### 19. (Optional) Running a sample GPU application

> [More Info](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/install-gpu-ocp.html#running-a-sample-gpu-application)

- Run a simple CUDA VectorAdd sample, which adds two vectors together to ensure the GPUs have bootstrapped correctly
- Create a test project

  - ```sh
    oc new-project sandbox
    ```

    ```sh
    # expected output
    Now using project "sandbox" on server "https://api.cluster-582gr.582gr.sandbox2642.opentlc.com:6443".
    ```

- Create the sample app

  - ```sh
    oc create -f configs/nvidia-gpu-sample-app.yaml
    ```

    ```sh
    # expected output
    pod/cuda-vectoradd created
    ```

- Check the logs of the container

  - ```sh
    oc logs cuda-vectoradd
    ```

    ```sh
    # expected output
    [Vector addition of 50000 elements]
    Copy input data from the host memory to the CUDA device
    CUDA kernel launch with 196 blocks of 256 threads
    Copy output data from the CUDA device to the host memory
    Test PASSED
    Done
    ```

  - ```sh
    oc get pods
    ```

    ```sh
    # expected output
    oc get pods
    NAME             READY   STATUS      RESTARTS   AGE
    cuda-vectoradd   0/1     Completed   0          54s
    ```

- Get information about the GPU

  - ```sh
    oc project nvidia-gpu-operator
    ```

- View the new pods

  - ```sh
    oc get pod -o wide -l openshift.driver-toolkit=true
    ```

    ```sh
    # expected output
    NAME                                                  READY   STATUS    RESTARTS   AGE   IP            NODE                                       NOMINATED NODE   READINESS GATES
    nvidia-driver-daemonset-415.92.202407091355-0-64sml   2/2     Running   2          21h   10.130.0.8    ip-10-0-22-25.us-east-2.compute.internal   <none>           <none>
    nvidia-driver-daemonset-415.92.202407091355-0-clp7f   2/2     Running   2          21h   10.129.0.10   ip-10-0-22-15.us-east-2.compute.internal   <none>           <none>
    ```

- With the Pod and node name, run the nvidia-smi on the correct node.

  - ```sh
    oc exec -it nvidia-driver-daemonset-410.84.202203290245-0-xxgdv -- nvidia-smi
    ```

    ```sh
    # expected output
    Fri Jul 26 20:06:33 2024
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 550.54.15              Driver Version: 550.54.15      CUDA Version: 12.4     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  Tesla T4                       On  |   00000000:00:1E.0 Off |                    0 |
    | N/A   34C    P8             14W /   70W |       0MiB /  15360MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+

    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+
    ```

> - The first table reflects the information about all available GPUs (the example shows one GPU).
> - The second table provides details on the processes using the GPUs.

### 20. Enable GPU monitoring dashboard

> [More Info](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/enable-gpu-monitoring-dashboard.html)

- Download the latest NVIDIA DCGM Exporter Dashboard from the DCGM Exporter repository on GitHub:

  - ```sh
    curl -Lf https://github.com/NVIDIA/dcgm-exporter/raw/main/grafana/dcgm-exporter-dashboard.json -o scratch/dcgm-exporter-dashboard.json
    ```

    ```sh
    # expected output
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed
      0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
    100 18114  100 18114    0     0  23496      0 --:--:-- --:--:-- --:--:-- 23496
    ```

- Check for modifications

  - ```sh
    diff -u configs/files/nvidia-dcgm-dashboard.json scratch/dcgm-exporter-dashboard.json
    ```

    ```sh
    # expected output
    <blank>
    ```

- Create a config map from the downloaded file in the openshift-config-managed namespace

  - ```sh
    oc create -f configs/nvidia-dcgm-dashboard-cm.yaml
    ```

    ```sh
    # expected output
    configmap/nvidia-dcgm-exporter-dashboard created
    ```

- Label the config map to expose the dashboard in the Administrator perspective of the web console `dashboard`:

  - ```sh
    oc label configmap nvidia-dcgm-exporter-dashboard -n openshift-config-managed "console.openshift.io/dashboard=true"
    ```

    ```sh
    # expected output
    configmap/nvidia-dcgm-exporter-dashboard labeled
    ```

- (Optional) Label the config map to expose the dashboard in the Developer perspective of the web console `odc-dashboard`:

  - ```sh
    oc label configmap nvidia-dcgm-exporter-dashboard -n openshift-config-managed "console.openshift.io/odc-dashboard=true"
    ```

    ```sh
    # expected output
    configmap/nvidia-dcgm-exporter-dashboard labeled
    ```

- View the created resource and verify the labels

  - ```sh
    oc -n openshift-config-managed get cm nvidia-dcgm-exporter-dashboard --show-labels
    ```

    ```sh
    # expected output
    NAME                             DATA   AGE     LABELS
    nvidia-dcgm-exporter-dashboard   1      3m28s   console.openshift.io/dashboard=true,console.openshift.io/odc-dashboard=true
    ```

> View the NVIDIA DCGM Exporter Dashboard from the OCP UI from Administrator and Developer

### 21. Installing the NVIDIA GPU administration dashboard

> [More Info](https://docs.openshift.com/container-platform/4.12/observability/monitoring/nvidia-gpu-admin-dashboard.html)

- Add the Helm repository

  - ```sh
    helm repo add rh-ecosystem-edge https://rh-ecosystem-edge.github.io/console-plugin-nvidia-gpu
    ```

    ```sh
    # expected output
    "rh-ecosystem-edge" has been added to your repositories

    # possible output
    "rh-ecosystem-edge" already exists with the same configuration, skipping
    ```

- Helm update

  - ```sh
    helm repo update
    ```

    ```sh
    # expected output
    Hang tight while we grab the latest from your chart repositories...
    ...Successfully got an update from the "rh-ecosystem-edge" chart repository
    <snipped>
    Update Complete. ⎈Happy Helming!⎈
    ```

- Install the Helm chart in the default NVIDIA GPU operator namespace

  - ```sh
    helm install -n nvidia-gpu-operator console-plugin-nvidia-gpu rh-ecosystem-edge/console-plugin-nvidia-gpu
    ```

    ```sh
    # expected output
    NAME: console-plugin-nvidia-gpu
    LAST DEPLOYED: Tue Jul 16 18:28:55 2024
    NAMESPACE: nvidia-gpu-operator
    STATUS: deployed
    REVISION: 1
    NOTES:

    <snipped>

    ...
    ```

- Check if a plugins field is specified

  - ```sh
    oc get consoles.operator.openshift.io cluster --output=jsonpath="{.spec.plugins}"
    ```

    ```sh
    # expected output
    <blank>
    ```

- If not, then run the following to enable the plugin

  - ```sh
    oc patch consoles.operator.openshift.io cluster --patch '[{"op": "add", "path": "/spec/plugins/-", "value": "console-plugin-nvidia-gpu" }]' --type=json
    ```

    ```sh
    # expected output
    console.operator.openshift.io/cluster patched
    ```

- Add the required DCGM Exporter metrics ConfigMap to the existing NVIDIA operator ClusterPolicy CR

  - ```sh
    oc patch clusterpolicies.nvidia.com gpu-cluster-policy --patch '{ "spec": { "dcgmExporter": { "config": { "name": "console-plugin-nvidia-gpu" } } } }' --type=merge
    ```

    ```sh
    # expected output
    clusterpolicy.nvidia.com/gpu-cluster-policy patched
    ```

You should receive a message on the console "Web console update is available" > Refresh the web console.

#### Go to Compute > GPUs

> Notice the `Telsa T4 Single-Instance` at the top of the screen. This GPU is not shareable (i.e. sliced, partitioned, fractioned) yet.

> The dashboard relies mostly on Prometheus metrics exposed by the NVIDIA DCGM Exporter, but the default exposed metrics are not enough for the dashboard to render the required gauges. Therefore, the DGCM exporter is configured to expose a custom set of metrics, as shown here.

- ```sh
  oc get cm console-plugin-nvidia-gpu -n nvidia-gpu-operator -o yaml
  ```

  ```sh
  # expected output
  apiVersion: v1
  data:
    dcgm-metrics.csv: |
      DCGM_FI_PROF_GR_ENGINE_ACTIVE, gauge, gpu utilization.
      DCGM_FI_DEV_MEM_COPY_UTIL, gauge, mem utilization.
      DCGM_FI_DEV_ENC_UTIL, gauge, enc utilization.
      DCGM_FI_DEV_DEC_UTIL, gauge, dec utilization.
      DCGM_FI_DEV_POWER_USAGE, gauge, power usage.
      DCGM_FI_DEV_POWER_MGMT_LIMIT_MAX, gauge, power mgmt limit.
      DCGM_FI_DEV_GPU_TEMP, gauge, gpu temp.
      DCGM_FI_DEV_SM_CLOCK, gauge, sm clock.
      DCGM_FI_DEV_MAX_SM_CLOCK, gauge, max sm clock.
      DCGM_FI_DEV_MEM_CLOCK, gauge, mem clock.
      DCGM_FI_DEV_MAX_MEM_CLOCK, gauge, max mem clock.
  kind: ConfigMap
  metadata:
    annotations:
      meta.helm.sh/release-name: console-plugin-nvidia-gpu
      meta.helm.sh/release-namespace: nvidia-gpu-operator
    ...
    labels:
      app.kubernetes.io/component: console-plugin-nvidia-gpu
      app.kubernetes.io/instance: console-plugin-nvidia-gpu
      app.kubernetes.io/managed-by: Helm
      app.kubernetes.io/name: console-plugin-nvidia-gpu
      app.kubernetes.io/part-of: console-plugin-nvidia-gpu
      app.kubernetes.io/version: latest
      helm.sh/chart: console-plugin-nvidia-gpu-0.2.4
    name: console-plugin-nvidia-gpu
    namespace: nvidia-gpu-operator
    ...
  ```

- View the deployed resources

  - ```sh
    oc -n nvidia-gpu-operator get all -l app.kubernetes.io/name=console-plugin-nvidia-gpu
    ```

### GPU sharing methods

> Why? By default, you get one workload per GPU. This is inefficient for certain use cases. [How can you share a GPU to 1:N workloads](https://docs.openshift.com/container-platform/4.15/architecture/nvidia-gpu-architecture-overview.html#nvidia-gpu-prerequisites_nvidia-gpu-architecture-overview):

> For NVIDIA:

> - [Time-slicing NVIDIA GPUs](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/time-slicing-gpus-in-openshift.html#)
> - [Multi-Instance GPU (MIG)](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/mig-ocp.html)
> - [NVIDIA vGPUs](https://docs.nvidia.com/datacenter/cloud-native/openshift/23.9.2/nvaie-with-ocp.html?highlight=passthrough#openshift-container-platform-on-vmware-vsphere-with-nvidia-vgpus)

### 22. Configuring GPUs with time slicing

> [More Info](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/time-slicing-gpus-in-openshift.html#configuring-gpus-with-time-slicing)

> The following sections show you how to configure NVIDIA Tesla T4 GPUs, as they do not support MIG, but can easily accept multiple small jobs.

> The NVIDIA GPU Operator enables oversubscription of GPUs through a set of extended options for the NVIDIA Kubernetes Device Plugin. GPU time-slicing enables workloads that are scheduled on oversubscribed GPUs to interleave with one another.

> You configure GPU time-slicing by performing the following high-level steps:

> 1. Add a config map to the namespace that is used by the GPU operator (i.e. `device-plugin-config`).
> 2. Configure the cluster policy so that the device plugin uses the config map. (i.e. `gpu-cluster-policy`)
> 3. Apply a label to the nodes that you want to configure for GPU time-slicing. (i.e. `Tesla-T4-SHARED`)

> Important: The ConfigMap name is `device-plugin-config` and the profile name is `time-sliced-8`. These names are important as you can change them for your purposes. You can list multiple GPU profiles like in the [ai-gitops-catalog](https://github.com/redhat-na-ssa/demo-ai-gitops-catalog/blob/main/components/operators/gpu-operator-certified/instance/components/time-sliced-4/patch-device-plugin-config.yaml). Also, [see Applying multi-node configuration](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html#applying-multiple-node-specific-configurations)

> On a machine with one GPU, the following config map configures Kubernetes so that the node advertises 8 GPU resources. A machine with two GPUs advertises 16 GPUs, and so on. [More Info](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html#about-configuring-gpu-time-slicing)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
  namespace: nvidia-gpu-operator
data:
  time-sliced-8: |-
    version: v1
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 8
```

- Apply the device plugin configuration

  - ```sh
    oc apply -f configs/nvidia-gpu-deviceplugin-cm.yaml
    ```

    ```sh
    # expected output
    configmap/device-plugin-config created
    ```

- Tell the GPU Operator which ConfigMap, in this case `device-plugin-config` to use for the device plugin configuration.

  - ```sh
    oc patch clusterpolicy gpu-cluster-policy \
        -n nvidia-gpu-operator --type merge \
        -p '{"spec": {"devicePlugin": {"config": {"name": "device-plugin-config"}}}}'
    ```

    ```sh
    # expected output
    clusterpolicy.nvidia.com/gpu-cluster-policy patched
    ```

- Apply the configuration to all the nodes you have with Tesla T GPUs. GFD, labels the nodes with the GPU product, in this example Tesla-T4, so you can use a node selector to label all of the nodes at once.

  - ```sh
    oc label --overwrite node \
        --selector=nvidia.com/gpu.product=Tesla-T4 \
        nvidia.com/device-plugin.config=time-sliced-8
    ```

    ```sh
    # expected output
    node/ip-10-0-29-207.us-east-2.compute.internal labeled
    node/ip-10-0-36-189.us-east-2.compute.internal labeled
    ```

- Patch the NVIDIA GPU Operator ClusterPolicy to use the timeslicing configuration by default.

  - ```sh
    oc patch clusterpolicy gpu-cluster-policy \
        -n nvidia-gpu-operator --type merge \
        -p '{"spec": {"devicePlugin": {"config": {"default": "time-sliced-8"}}}}'
    ```

    ```sh
    # expected output
    clusterpolicy.nvidia.com/gpu-cluster-policy patched
    ```

- The applied configuration creates eight replicas for Tesla T4 GPUs, so the nvidia.com/gpu external resource is set to 8. You can apply a cluster-wide default time-slicing configuration. You can also apply node-specific configurations. For example, you can apply a time-slicing configuration to nodes with Tesla-T4 GPUs only and not modify nodes with other GPU models.

  - ```sh
    oc get node --selector=nvidia.com/gpu.product=Tesla-T4-SHARED -o json | jq '.items[0].status.capacity'
    ```

    The `-SHARED` product name suffix ensures that you can specify a node selector to assign pods to nodes with time-sliced GPUs.

    ```sh
    # expected output
    {
      "cpu": "16",
      "ephemeral-storage": "104266732Ki",
      "hugepages-1Gi": "0",
      "hugepages-2Mi": "0",
      "memory": "65029276Ki",
      "nvidia.com/gpu": "8",
      "pods": "250"
    }
    ```

- Verify that GFD labels have been added to indicate time-sharing.

  1. The `nvidia.com/gpu.count` label reports the number of physical GPUs in the machine.
  2. The `nvidia.com/gpu.product` label includes a `-SHARED` suffix to the product name.
  3. The `nvidia.com/gpu.replicas` label matches the reported capacity.

  - ```sh
    oc get node --selector=nvidia.com/gpu.product=Tesla-T4-SHARED -o json \
    | jq '.items[0].metadata.labels' | grep nvidia
    ```

    ```sh
    # expected output
      ...
      "nvidia.com/gpu.count": "1",
      ...
      "nvidia.com/gpu.product": "Tesla-T4-SHARED",
      "nvidia.com/gpu.replicas": "8",
      ...
    ```

### 23. Configure Taints and Tolerations

> Why? Prevent non-GPU workloads from being scheduled on the GPU nodes.

- Taint the GPU nodes with `nvidia.com/gpu`. This MUST match the Accelerator profile taint key you use (this could be different, i.e. `nvidia-gpu-only`).

  - ```sh
    oc adm taint node -l nvidia.com/gpu.machine nvidia.com/gpu=:NoSchedule --overwrite
    ```

- Edit the `ClusterPolicy` in the NVIDIA GPU Operator under the `nvidia-gpu-operator` project. Add the below section to `.spec.daemonsets:`

  - ```sh
    oc edit ClusterPolicy
    ```

    ```sh
      daemonsets:
        tolerations:
        - effect: NoSchedule
          operator: Exists
          key: nvidia.com/gpu
    ```

- Cordon the GPU node, drain the GPU tainted nodes and terminate workloads

  - ```sh
    oc adm drain -l nvidia.com/gpu.machine --ignore-daemonsets --delete-emptydir-data
    ```

- Allow the GPU node to be schedulable again per tolerations

  - ```sh
    oc adm uncordon -l nvidia.com/gpu.machine
    ```

- Get the name of the gpu node

  - ```sh
    MACHINE_SET_TYPE=$(oc get machineset -n openshift-machine-api -o name |  egrep gpu)
    ```

- Taint the machineset for any new nodes that get added to be tainted with `nvidia.com/gpu`

  - ```sh
    oc -n openshift-machine-api \
      patch "${MACHINE_SET_TYPE}" \
      --type=merge --patch '{"spec":{"template":{"spec":{"taints":[{"key":"nvidia.com/gpu","value":"","effect":"NoSchedule"}]}}}}'
    ```

> Tolerations will be set in the RHOAI accelerator profiles that match the Taint key.

### 24. (Optional) Configuring the cluster autoscaler

> [More Info](https://docs.openshift.com/container-platform/4.15/machine_management/applying-autoscaling.html)

## Configuring distributed workloads

> [More Info](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.10/html/working_with_distributed_workloads/configuring-distributed-workloads_distributed-workloads)

> Components required for Distributed Workloads

> 1. dashboard
> 1. workbenches
> 1. datasciencepipelines
> 1. codeflare
> 1. kueue
> 1. ray

- Verify the necessary pods are running - When the status of the codeflare-operator-manager-[pod-id], kuberay-operator-[pod-id], and kueue-controller-manager-[pod-id] pods is Running, the pods are ready to use.

  - ```sh
    oc get pods -n redhat-ods-applications | grep -E 'codeflare|kuberay|kueue'
    ```

    ```sh
    # expected output
    codeflare-operator-manager-6bbff698d-74fpz                        1/1     Running   7 (107m ago)   21h
    kuberay-operator-bf97858f4-zg45s                                  1/1     Running   8 (10m ago)    21h
    kueue-controller-manager-77c758b595-hgrz7                         1/1     Running   8 (10m ago)    21h
    ```

### 25. Configuring quota management for distributed workloads

- Create an empty Kueue resource flavor

  > Why? Resources in a cluster are typically not homogeneous. A ResourceFlavor is an object that represents these resource variations (i.e. Nvidia A100 versus T4 GPUs) and allows you to associate them with cluster nodes through labels, taints and tolerations.

- Apply the rhoai kueue resource flavor configuration to create the `default-flavor`

  - ```sh
    oc apply -f configs/rhoai-kueue-default-flavor.yaml
    ```

    ```sh
    # expected output
    resourceflavor.kueue.x-k8s.io/default-flavor created
    ```

- Create a cluster queue to manage the empty Kueue resource flavor

  > Why? A ClusterQueue is a cluster-scoped object that governs a pool of resources such as pods, CPU, memory, and hardware accelerators. Only batch administrators should create ClusterQueue objects.

  > What is this cluster-queue doing? This ClusterQueue admits Workloads if and only if:

  > - The sum of the CPU requests is less than or equal to 9.
  > - The sum of the memory requests is less than or equal to 36Gi.
  > - The total number of pods is less than or equal to 5.

  > ![IMPORTANT]
  > Replace the example quota values (9 CPUs, 36 GiB memory, and 5 NVIDIA GPUs) with the appropriate values for your cluster queue. The cluster queue will start a distributed workload only if the total required resources are within these quota limits. Only homogenous NVIDIA GPUs are supported.

- Apply the configuration to create the `cluster-queue`

  - ```sh
    oc apply -f configs/rhoai-kueue-cluster-queue.yaml
    ```

    ```sh
    # expected output
    clusterqueue.kueue.x-k8s.io/cluster-queue created
    ```

- Create a local queue that points to your cluster queue

  > Why? A LocalQueue is a namespaced object that groups closely related Workloads that belong to a single namespace. Users submit jobs to a LocalQueue, instead of to a ClusterQueue directly.

  > ![NOTE]
  > Update the `name` and `namespace` accordingly.

- Apply the configuration to create the local-queue object

  - ```sh
    oc new-project sandbox
    oc apply -f configs/rhoai-kueue-local-queue.yaml
    ```

    ```sh
    # expected output
    localqueue.kueue.x-k8s.io/local-queue-test created
    ```

> How do users known what queues they can submit jobs to? Users submit jobs to a LocalQueue, instead of to a ClusterQueue directly. Tenants can discover which queues they can submit jobs to by listing the local queues in their namespace.

- Verify the local queue is created

  - ```sh
    oc get -n sandbox queues
    ```

    ```sh
    # expected output
    NAME               CLUSTERQUEUE    PENDING WORKLOADS   ADMITTED WORKLOADS
    local-queue-test   cluster-queue   0
    ```

### 26. (Optional) Configuring the CodeFlare Operator

- Get the `codeflare-operator-config` configmap

  - ```sh
    oc get cm codeflare-operator-config -n redhat-ods-applications -o yaml
    ```

> In the `codeflare-operator-config`, data:config.yaml:kuberay section, you can patch the [following](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.10/html/working_with_distributed_workloads/configuring-distributed-workloads_distributed-workloads#configuring-the-codeflare-operator_distributed-workloads)

> 1.  ingressDomain option is null (ingressDomain: "") by default.
> 1.  mTLSEnabled option is enabled (mTLSEnabled: true) by default.
> 1.  rayDashboardOauthEnabled option is enabled (rayDashboardOAuthEnabled: true) by default.

- Recommended to keep default. If needed, apply the configuration to update the object

  - ```sh
    oc apply -f configs/rhoai-codeflare-operator-config.yaml
    ```

## Administrative Configurations for RHOAI

### Review RHOAI Dashboard Settings

Access the RHOAI Dashboard > Settings.

#### Notebook Images

- Import new notebook images

#### Cluster Settings

- Model serving platforms
- PVC size (see Backing up data)
- Stop idle notebooks
- Usage data collection
- Notebook pod tolerations

#### Accelerator Profiles

- Manage accelerator profile settings for users in your organization

### 27. Add a new Accelerator Profile

> [Enabling GPU support in OpenShift AI](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.10/html/installing_and_uninstalling_openshift_ai_self-managed/enabling-gpu-support_install)

> RHOAI dashboard and check the **Settings > Accelerator profiles** - There should be none listed.

Check the current configmap

- ```sh
  oc get cm migration-gpu-status -n redhat-ods-applications -o yaml
  ```

  ```sh
  # expected output
  apiVersion: v1
  data:
    migratedCompleted: "true"
  kind: ConfigMap
  metadata:
    creationTimestamp: "2024-07-16T17:43:10Z"
    name: migration-gpu-status
    namespace: redhat-ods-applications
    resourceVersion: "48442"
    uid: 1724c41a-8bbc-4619-a55f-df029d98f2ff
  ```

- Delete the migration-gpu-status ConfigMap

  - ```sh
    oc delete cm migration-gpu-status -n redhat-ods-applications
    ```

    ```sh
    # expected output
    configmap "migration-gpu-status" deleted
    ```

- Restart the dashboard replicaset

  - ```sh
    oc rollout restart deployment rhods-dashboard -n redhat-ods-applications
    ```

    ```sh
    # expected output
    deployment.apps/rhods-dashboard restarted
    ```

- Wait until the Status column indicates that all pods in the rollout have fully restarted

  - ```sh
    oc get pods -n redhat-ods-applications | egrep rhods-dashboard
    ```

    ```sh
    # expected output
    rhods-dashboard-69b9bc879d-k6gzb                                  2/2     Running   6                25h
    rhods-dashboard-7b67c58d9b-4xzr7                                  2/2     Running   0                67s
    rhods-dashboard-7b67c58d9b-chgln                                  2/2     Running   0                67s
    rhods-dashboard-7b67c58d9b-dk8sx                                  0/2     Running   0                7s
    rhods-dashboard-7b67c58d9b-tsngh                                  2/2     Running   0                67s
    rhods-dashboard-7b67c58d9b-x5v89                                  0/2     Running   0                7s
    ```

- Refresh the RHOAI dashboard and check the **Settings > Accelerator profiles** - There should be `NVIDIA GPU` enabled.

- Check the acceleratorprofiles

  - ```sh
    oc get acceleratorprofile -n redhat-ods-applications
    ```

    ```sh
    # expected output
    NAME           AGE
    migrated-gpu   83s
    ```

- Review the acceleratorprofile configuration

  - ```sh
    oc describe acceleratorprofile -n redhat-ods-applications
    ```

    ```sh
    # expected output
    Name:         migrated-gpu
    Namespace:    redhat-ods-applications
    Labels:       <none>
    Annotations:  <none>
    API Version:  dashboard.opendatahub.io/v1
    Kind:         AcceleratorProfile
    Metadata:
      Creation Timestamp:  2024-07-17T19:28:24Z
      Generation:          1
      Resource Version:    609012
      UID:                 8f64a27f-6593-43a6-873d-4796e920494f
    Spec:
      Display Name:  NVIDIA GPU
      Enabled:       true
      Identifier:    nvidia.com/gpu
      Tolerations:
        Effect:    NoSchedule
        Key:       nvidia.com/gpu
        Operator:  Exists
    Events:        <none>
    ```

- Verify the `taints` key set in your Node / MachineSets match your `Accelerator Profile`.

### 28. Add serving runtime

> Serving Runtimes

> - Single-model serving platform
>   - Caikit TGIS ServingRuntime for KServe
>   - OpenVINO Model Server
>   - TGIS Standalone ServingRuntime for KServe
> - Multi-model serving platform
>   - OpenVINO Model Server

- From RHOAI, Settings > Serving runtimes > Click Add Serving Runtime.

  - Option 1:

    - Select `Multi-model serving`
    - Select `Start from scratch`
    - Review, Copy and Paste in the content from `configs/rhoai-add-serving-runtime.yaml`
    - Add and confirm the runtime can be selected in a Data Science Project

  - Option 2:

    - Review `configs/rhoai-add-serving-runtime-template.yaml`
    - `oc apply -f configs/rhoai-add-serving-runtime-template.yaml -n redhat-ods-applications`
    - Add and confirm the runtime can be selected in a Data Science Project

### 29. Review Backing up data

> Refer to [A Guide to High Availability / Disaster Recovery for Applications on OpenShift](https://www.redhat.com/en/blog/a-guide-to-high-availability/disaster-recovery-for-applications-on-openshift)

#### Control plane backup and restore operations

> You must [back up etcd](https://docs.openshift.com/container-platform/4.15/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html#backup-etcd) data before shutting down a cluster; etcd is the key-value store for OpenShift Container Platform, which persists the state of all resource objects.

#### Application backup and restore operations

> The OpenShift API for Data Protection (OADP) product safeguards customer applications on OpenShift Container Platform. It offers comprehensive disaster recovery protection, covering OpenShift Container Platform applications, application-related cluster resources, persistent volumes, and internal images. OADP is also capable of backing up both containerized applications and virtual machines (VMs).

> However, OADP does not serve as a disaster recovery solution for [etcd](https://docs.openshift.com/container-platform/4.15/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html#backup-etcd) or OpenShift Operators.

## Answer key

The following command will apply all configurations from the checklist procedure above.

```sh
# setup machinesets for autoscaling
. configs/functions.sh
ocp_aws_cluster_autoscaling

# apply all configs (still need to patch things)
until oc apply -f configs; do : ; done
```

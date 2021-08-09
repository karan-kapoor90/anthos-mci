# Objectives

> The following guide has been tested on Google CloudShell and may not work happily on Windows or Mac. Cloud Shell is FREE, so there really isn't any good reason to not use it ;)

The objective is create a service mesh using multiple kubernetes clusters distributed around the globe, to host services for customers to consume. Here are a couple of things that should happen:

- Users coming from India should be redirected to the pods running closest to them. Users coming from Australia should be redirected to the clusters in Australia.
- Routing of services should be defined using a single ingress with a single public global IP address for internet incoming requests. 
- Service Discovery and inter-cluster load-balancing should happen. If a service deployed across both the clusters is called by another service within either of the clusters, it should get responses from the common service deployed across both the clusters.

Scenario's 1 and 2 can be accomplished using a Multi-cluster Ingress setup on GKE. Scenario 3 requires the use of Anthos Service mesh deployed across both the workload clusters. We need to meet the following Objectives:

**Objective 1** - Setup multi-cluster ingress that uses a Global LB for load balancing and can do intelligent routing
**Objective 2** - Setup multi-cluster mesh using ASM


> A combination of both the Objectives is possible by using Multi-Cluster Ingress and Anthos Service Mesh together, but that is out of the scope of this document.

# Setup

## Assumptions

- Both the clusters are in the same GCP project
- Both the clusters are in the same GCP VPC
- Only the app-tier is considered for this demonstration
- This is not a production grade deployment by any means. This is only for experimenting and experiencing some of the capabilities offered by Anthos Service Mesh on Google Cloud. 
- Workload Identity will be used to enable access between GCP resources instead of using GCP Service Accounts. 

## Environment Setup

> This step is common across all 3 scenarios.

1. Create/ get access to a Project on GCP where you are the Administrator/ Owner. On the top left corner of the page, click on the project name, and copy the project ID.

2. Setup the Environment with a few variable that will be used extensively throughout.

    ```bash
    export PROJECT_ID=<your-project-id>
    export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
    export MESH_ID="proj-${PROJECT_ID}"
    export C1=cluster1   # name of the first cluster
    export R1=asia-southeast1   # Region for the first cluster to be deployed in
    export C2=cluster2   # name of the second cluster
    export R2=australia-southeast1   # Region for the se cluster to be deployed in
    mkdir asm-setup && export PROJECT_DIR=${PWD}/asm-setup

    gcloud config set project ${PROJECT_ID}
    ```
3. Enable the Google Cloud API's that will be required for these scenarios:

    ```bash
    gcloud services enable \
    --project=PROJECT_ID \
    anthos.googleapis.com \
    multiclusteringress.googleapis.com \
    container.googleapis.com \
    compute.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    cloudtrace.googleapis.com \
    meshca.googleapis.com \
    meshtelemetry.googleapis.com \
    meshconfig.googleapis.com \
    iamcredentials.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    cloudresourcemanager.googleapis.com \
    stackdriver.googleapis.com
    ```

4. Enable kubectl alias and auto completion - because, why not simplify when one can :)

    ```bash
    echo alias k=kubectl >> ~/.bash_profile
    echo source <(kubectl completion bash) >> ~/.bash_profile
    echo complete -F __start_kubectl k >> ~/.bash_profile
    ```

## GKE Cluster Setup

> Providing the zone while creating a cluster will make it a zonal cluster. If you want better HA and the wokrer nodes for your clusters to be distributed across the entire region (which hsa 3 or more zones), use the `--region` switch to create a regional cluster.

1. Create 2 clusters in 2 different regions. DO NOT MISS ANY OF THE FLAGS MENTIONED IN THE COMMAND
    
    ```bash
    gcloud config set project ${PROJECT_ID}
    gcloud container clusters create ${C1} --region ${R1} --enable-ip-alias --workload-pool=${WORKLOAD_POOL} --mesh-id=${MESH_ID}
    gcloud container clusters create ${C2} --region ${R2} --enable-ip-alias --workload-pool=${WORKLOAD_POOL} --mesh-id=${MESH_ID}
    ```

    > Note: If you're using pre-existing, pre-created clusters, you need to ensure that IP Aliases are enabled. On the GCP console, navigate to your cluster and check if VPC-native traffic Routing is enabled on the cluster. 

    > Workload Identity is going to be used by Multi-cluster ingress to enable communication b/w the clusters. Check if the clusters you intend to setup for this demo have it enabled, by looking for the Workload Identity Feature to be enabled on the cluster. If it is not enabled,

    ```bash
    gcloud container clusters update <CLUSTER_NAME> \
    --project=${PROJECT_ID} \
    --workload-pool=${WORKLOAD_POOL} \
    --region <cluster-region>
    ```

    > For ASM to work, your clusters need to be labelled with the `mesh-id` label. In case your clusters already exist and you only want to add the label to an existing cluster,

    ```bash
    gcloud container clusters update <CLUSTER_NAME> \
    --project=${PROJECT_ID} \
    --update-labels=mesh_id=${MESH_ID} \
    --region <cluster-region>
    ```

2. Add the credentials to access the clusters we've just created.

    ```bash
    gcloud container clusters get-credentials ${C1} --region ${R1}
    gcloud container clusters get-credentials ${C2} --region ${R2}
    ```

3. Make the context names more human readable

    ```bash
    kubectl config rename-context gke_${PROJECT_ID}_${R1}_${C1} ${C1}
    kubectl config rename-context gke_${PROJECT_ID}_${R2}_${C2} ${C2}
    ```

# Objective 1: Multi-cluster ingress/ Global LoadBalancing

> You do not need Anthos Service Mesh or Istio in order for this to work. It uses something known as [Network Endpoint Groups](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg#overview) to route traffic from the LB directly to the Pods.


In order for Global Load Balancing to work, GCP can provide us with a random Static IP address that is ephemeral to the life of the Load Balancer. When the load balancer, is deleted, the static Ip address will also be released. Optionally, one can create a static IP reservation, which will give us a static IP which will not be released till one doesn't manually delete it. We can then use this static IP as the Load balancer IP and connect it to the FQDN for our eventual website.

> IMPORTANT: The way Anthos for multi-cluster works, is by adding clusters to a hub (this is the gkehub api we enabled earlier). For Multi-cluster Ingress, while ALL THE CLUSTERS where your workloads will be running, the data plane, need to be added to the hub, but ONLY ONE of the clusters must be made the Configuration Cluster. Keep this in mind when configuring the Configuration Cluster.

1. Create a global static Ip address for the Global LB (HTTP LB)

    ```bash
    gcloud compute addresses create glb-ip --global

    # Note the external IP address created here. If you're not able to see the IP, wait for a couple of minutes before trying
    gcloud compute addresses list --global 

    export GCLB_IP=<paste-static-ip-here>   # export the static IP you created as an environment variable.

    ```

2. Create a DNS entry that links the URL that users will use to access your application, with the GCLB_IP. Note that this creates a cloud Endpoint, and does not use the Cloud DNS service from GCP. 

    > YAML is extremely sensitive to spaces, so be mindful of the formatting. 

    ```bash
    cat <<EOF > dns-spec.yaml
    swagger: "2.0"
    info:
        description: "Cloud Endpoints DNS"
        title: "Cloud Endpoints DNS"
        version: "1.0.0"
    paths: {}
    host: "frontend.endpoints.${PROJECT_ID}.cloud.goog"
    x-google-endpoints:
    - name: "frontend.endpoints.${PROJECT_ID}.cloud.goog"
        target: "${GCLB_IP}"
    EOF

    gcloud endpoints services deploy dns-spec.yaml
    ```


    Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress-setup#requirements_for 


3. The clusters need to be added to the hub.   


    ```bash
    # Adding the first cluster to the hub
    gcloud container hub memberships register ${C1} \
        --project=${PROJECT_ID} \
        --gke-uri=${R1}/${C1} \
        --enable-workload-identity

    # Adding the second cluster to the hub
    gcloud container hub memberships register ${C2} \
        --project=${PROJECT_ID} \
        --gke-uri=${R2}/${C2} \
        --enable-workload-identity

    # Confirm that both the clusters have been added to the hub
    gcloud container hub memberships list --project={PROJECT_ID}
    ```

    Both the clusters should be visible as members of the hub.

4. Once done, we have to select one of the clusters as the config cluster. This will house the MultiClusterIngress and MultiClusterService resources. **Only 1 cluster in a fleet can be an active config cluster**

    ```bash
    gcloud beta container hub ingress enable \
    --config-membership=projects/${PROJECT_ID}/locations/global/memberships/${C1} 
   
    # Confirm cluster registration as the config cluster
    gcloud beta container hub ingress describe
    ```

    If all is running well, we can now deploy the zoneprinter application, and test the setup.

5. Creating the application deployment and namespace resources **ON BOTH CLUSTERS**

    Create a file called zoneprinter.yaml and write the following contents into it

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: zoneprinter
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: zone-ingress
    namespace: zoneprinter
    labels:
        app: zoneprinter
    spec:
    selector:
        matchLabels:
        app: zoneprinter
    template:
        metadata:
        labels:
            app: zoneprinter
        spec:
        containers:
        - name: frontend
            image: gcr.io/google-samples/zone-printer:0.2
            ports:
            - containerPort: 8080
    ```

    Apply them to the clusters

    ```bash
    k apply -f zoneprinter.yaml --context ${C1}
    k apply -f zoneprinter.yaml --context ${C2}
    ```

6. Verify that all the pods are running in the namespace in both the clusters.

    ```bash
    k get deploy -n zoneprinter --context ${C1}
    k get deploy -n zoneprinter --context ${C2}
    ```

7. On the **Config cluster**, installing the Multi-cluster Service. This will create a headless cluster IP service for the zoneprinter application in both the clusters.

    Create a yaml file called `mcs.yaml` with the following contents

    ```yaml
    apiVersion: networking.gke.io/v1
    kind: MultiClusterService
    metadata:
        name: zone-mcs
        namespace: zoneprinter
    spec:
    template:
        spec:
        selector:
            app: zoneprinter
        ports:
        - name: web
            protocol: TCP
            port: 8080
            targetPort: 8080
    ```

    ```bash
    #  Switch context to the config cluster
    k config use-context ${C1}
    k apply -f mcs.yaml
    ```

8. Wait a couple of minutes and validate headless service creation in both the cluster. You should see a `ClusterIP` Service in each of the clusters. 

    ```bash
    k get svc -n zoneprinter --context ${C1}
    k get svc -n zoneprinter --context ${C2}
    ```

9. The moment of truth, deploying the `MultiClusterIngress` will create the HTTP(S) Loadbalancer service that will bring all of this together. 

    > This step also needs to be done in the config cluster, meaning context `${C1}`. Ensure you completed Step 1. of this objective, where you set the `GCLB_IP` environment variable.

    ```bash
    cat <<EOF > mci.yaml
    apiVersion: networking.gke.io/v1
    kind: MultiClusterIngress
    metadata:
        name: zone-ingress
        namespace: zoneprinter
        annotations:
          networking.gke.io/static-ip: ${GCLB_IP}
    spec:
    template:
        spec:
        backend:
            serviceName: zone-mcs
            servicePort: 8080
    EOF

    k apply -f mci.yaml
    ```

10. Wait a couple of minutes while the configuration gets setup. There are many ways to check if the configuration has completed successfully. 

    ```bash
    k get mci zone-printer -n zoneprinter -o=jsonpath={.status.VIP}   # this should prin the static IP we created earlier, meaning that the app is deployed.

    k describe mci zone-printer -n zoneprinter   # Use this to see the events and debug if the previous command doesn't print the VIP.
    ```

11. The website should now be available at 

    ```bash
    gcloud endpoints services list --format="value(serviceName)"
    ```

    > If the website is not av
12. Validating - Test the services from different regions using your local broswer as well as https://geotargetly.com/geo-browse 

# Objective 2: Multi-cluster mesh setup using ASM

References: https://cloud.google.com/service-mesh/docs/scripted-install/asm-onboarding then https://cloud.google.com/service-mesh/docs/scripted-install/gke-project-cluster-setup then https://cloud.google.com/service-mesh/docs/scripted-install/gke-install 
References: https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster 


## Downloading the binaries

1. Update your shell

    ```bash
    gcloud components update
    ```

2. Download the latest version of ASM and the it's SHA checksum

    ```bash
    curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.10 > install_asm
    curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.10.sha256 > install_asm.sha256
    sha256sum -c --ignore-missing install_asm.sha256
    chmod +x install_asm
    ```

    > Ensure that GCP API's were enabled as per the Environment Setup process. Technically though, you can install ASM on the clusters and not enable the Anthos API's as well, but that would hold back some of the other core Anthos features.

## Setting up Anthos Service Mesh

1. Validate and install ASM on both the clusters

    **Installing on the first cluster**

    ```bash
    ./install_asm \
    --project_id ${PROJECT_ID} \
    --cluster_name ${C1} \
    --cluster_location ${R1} \
    --mode install \
    --output_dir ${PROJECT_DIR} \
    --only_validate
    ```

    Install the default ASM features
    ```bash
    ./install_asm \
    --project_id ${PROJECT_ID} \
    --cluster_name ${C1} \
    --cluster_location ${R1} \
    --mode install \
    --ca mesh_ca \
    --output_dir ${PROJECT_DIR} \
    --enable_registration \
    --enable_all \
    --enable_cluster_labels
    ```

    **Installing on the second cluster**

    ```bash
    ./install_asm \
    --project_id ${PROJECT_ID} \
    --cluster_name ${C2} \
    --cluster_location ${R2} \
    --mode install \
    --output_dir ${PROJECT_DIR} \
    --only_validate
    ```

    Install the default ASM features
    ```bash
    ./install_asm \
    --project_id ${PROJECT_ID} \
    --cluster_name ${C2} \
    --cluster_location ${R2} \
    --mode install \
    --ca mesh_ca \
    --output_dir ${PROJECT_DIR} \
    --enable_registration \
    --enable_all \
    --enable_cluster_labels
    ```

2. Check istio and ASM namespaces installation in both the clusters. If all went well, both the clusters should have namespace called `istio-system` and `asm-system` and resources inside them.

    ```bash
    # Checking resources in Cluster1
    k get all -n istio-system --context ${C1}
    k get all -n asm-system --context ${C1}

    # Checking resources in Cluster2
    k get all -n istio-system --context ${C2}
    k get all -n asm-system --context ${C2}
    ```

3. Add `istioctl` CLI to PATH

    ```bash
    export PATH=${PWD}/asm-setup/istio-1.10.2-asm.3/bin:$PATH
    ```

4. In order for the ASM to load balance between pods deployed across both the clusters, trust needs to be established beween the 2 clusters. To do this, certificate for the first cluster needs to be installed in the second cluster and vice versa. This is called - [configuring endpoint discovery between clusters](https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster#verify_your_deployment)
    
    ```bash
    # Generating remote secrets for Cluster 1 and installing into Cluster 2
    istioctl x create-remote-secret --context=${C1} --name=${C1} | kubectl apply -f - --context=${C2}

    # Generating remote secrets for Cluster 2 and installing into Cluster 1
    istioctl x create-remote-secret --context=${C2} --name=${C2} | kubectl apply -f - --context=${C1}
    ```
    > Without this step, while a common service across the 2 clusters will be setup, loadbalancing b/w services deployed on both the clusters will not happen.

## Deploying an application to test ASM intra-cluster Load balancing

1. For applications deployed in the cluster to work with the service mesh, the Envoy sidecar would have to be deployed inside every pod for the application that we wish to do Service Mesh inside. In order for this to happen, sidecar injection will need to be enabled on the namespace in which the application is to be deployed. That is done by attaching a label on the namespace in which the application is going to be deployed. 

    We will need to get the value of the label that needs to be set on the namespaces. This can be run on either of the 2 clusters since both have ASM installed on them.

    ```bash
    k -n istio-system get po -l app=istiod --show-labels

    # Output 
        NAME                                 READY   STATUS    RESTARTS   AGE   LABELS
    istiod-asm-1102-3-76b8d69bf9-dzpmj   1/1     Running   0          30m   app=istiod,istio.io/rev=asm-1102-3,istio=istiod
    istiod-asm-1102-3-76b8d69bf9-zf94t   1/1     Running   0          30m   app=istiod,istio.io/rev=asm-1102-3,istio=istiod

    ```
    From the label column, note the value after `istio.io/rev=`, which is `asm-1102-3` in this case. Exporting it as a variable would make life easier. 

    ```bash
    export ASMV=asm-1102-3
    ```

2. Create a namespace for the test application and label it so that Envoy injection can happen when we deploy the application

    ```bash
    for CTX in ${C1} ${C2}
        do
            kubectl create --context=${CTX} namespace sample
            kubectl label --context=${CTX} namespace sample \
            istio-injection- istio.io/rev=${ASMV} --overwrite
        done
    ```

    You can check if the labelling has happened properly using the following command, and looking at the labels column in the output.

    ```bash
    k get ns sample --show-labels --context ${C1}
    k get ns sample --show-labels --context ${C2}
    ```
3. Deploy the `helloworld` kubernetes service to both the clusters, followed by the v1 and v2 deployment to individual clusters

    ```bash
    # deploying the service and the v1 deployment to the first cluster
    k create --context=${C1} -f ${PROJECT_DIR}/istio-1.10.2-asm.3/samples/helloworld/helloworld.yaml -l service=helloworld -n sample
    k create --context=${C1} -f ${PROJECT_DIR}/istio-1.10.2-asm.3/samples/helloworld/helloworld.yaml -l version=v1 -n sample

    # deploying the service and the v1 deployment to the first cluster
    k create --context=${C2} -f ${PROJECT_DIR}/istio-1.10.2-asm.3/samples/helloworld/helloworld.yaml -l service=helloworld -n sample
    k create --context=${C2} -f ${PROJECT_DIR}/istio-1.10.2-asm.3/samples/helloworld/helloworld.yaml -l version=v2 -n sample

    ```

## Testing intra-cluster load Balancing

The way we can test this is by creating a pod called `sleep` - which does exactly what the name suggests -, in both clusters, and use it to call the `helloworld` service. Since the helloworld service is being loadbalanced by ASM, the sleep service will get responses from both, cluster1 and cluster2 - which will show in the response as the version of the helloworld service. 

1. Deploying the sleep service in both the clusters

    ```bash
    for CTX in ${C1} ${C2}
    do
        kubectl apply --context=${CTX} \
        -f ${PROJECT_DIR}/istio-1.10.2-asm.3/samples/sleep/sleep.yaml -n sample
    done
    ```
    Test if the sleep pods have been deployed successfully by running the following 

    ```bash
    k -n sample get po,svc --context ${C1}
    k -n sample get po,svc --context ${C2}
    ```

2. Testing the outcome. The following command will run a curl command to call the helloworld service in the sample namespace, on port 5000 from inside the sleep container deployed on cluster1, and then do the same thing from the sleep containainer deployed in cluster2. Either way the desired outcome is that helloworld containers deployed on but the clusters (demarkated as v1 and v2), should both reply to the request.

    > You'll have to run the commnd for cluster 1 multiple time to see the responses coming from different versions, and the same for cluster 2.

    ```bash
    # Curl from sleep container in Cluster1
    kubectl exec -it --context="${C1}" -n sample -c sleep \
    "$(kubectl get pod --context="${C1}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- for i in `seq 1 20`; do curl -sS helloworld.sample:5000/hello; done

    # Curl from sleep container in Cluster2
    kubectl exec --context="${C2}" -n sample -c sleep \
    "$(kubectl get pod --context="${C2}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
    ```



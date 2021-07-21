# Objective

The objective is create a service mesh using multiple kubernetes clusters distributed around the globe, to host services for customers to consume. Here are a couple of things that should happen:
- Users coming from India should be redirected to the pods running closest to them. Users coming from Australia should be redirected to the clusters in Australia.
- Routing of services should be defined using a single ingress with a single global IP address
- Service Discovery and inter-cluster routing should be possible within the 2 clusters internally, without going over the public internet. 

You need to do 2 things
1. Setup multi-cluster mesh using ASM
2. Setup multi-cluster ingress that uses a Global LB for load balancing and can do intelligent routing

## Assumptions

- Both the clusters are in the same GCP project
- Both the clusters are in the same GCP VPC
- Only the app-tier is considered for this demonstration
- This is not a production grade deployment by any means. This is only for experimenting and experiencing some of the capabilities offered by Anthos Service Mesh on Google Cloud. 

# Multi-cluster mesh setup using ASM

References: https://cloud.google.com/service-mesh/docs/scripted-install/asm-onboarding then https://cloud.google.com/service-mesh/docs/scripted-install/gke-project-cluster-setup then https://cloud.google.com/service-mesh/docs/scripted-install/gke-install 
References: https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster 

- Create 2 clusters in 2 different regions. Do pass the `--enable-alias-ip`
    ```bash
    gcloud config set project PROJECT_NAME
    gcloud container clusters create cluster1 --region asia-southeast1 --enable-ip-alias
    gcloud container clusters create cluster2 --region australia-southeast1 --enable-ip-alias
    ```
- Install ASM on both the clusters

    First the Anthos API's need to be enabled. Do so by going to the following link after logging in: https://console.cloud.google.com/flows/enableapi?apiid=anthos.googleapis.com&_ga=2.55597547.199774792.1626845476-361436478.1626845476

    Then add the clusters you created earlier to. your Anthos cluster fleet from the UI.

    Then install ASM into the 2 clusters usign the following steps.

    Enable Cloud API's
    ```bash
    gcloud services enable \
    --project=PROJECT_ID \
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
    
    Download the ASM components and get ready to install them.

    ```bash
    gcloud components update
    curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.10 > install_asm
    curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.10.sha256 > install_asm.sha256
    sha256sum -c --ignore-missing install_asm.sha256
    chmod +x install_asm
    ```
    Validate the ASM installation before actually installing

    ```bash
    ./install_asm \
  --project_id PROJECT_ID \
  --cluster_name CLUSTER_NAME \
  --cluster_location CLUSTER_LOCATION \
  --mode install \
  --output_dir DIR_PATH \
  --only_validate
    ```

    Install the default ASM features
    ```bash
    ./install_asm \
  --project_id PROJECT_ID \
  --cluster_name CLUSTER_NAME \
  --cluster_location CLUSTER_LOCATION \
  --mode install \
  --ca mesh_ca \
  --output_dir DIR_PATH \
  --enable_registration \
  --enable_all
  --enable-registration
    ```

- Configure endpoint discovery between clusters - https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster#verify_your_deployment
    - Switch context into first cluster and apply the output of `istioctl x create-remote-secret` into the second cluster 
    - Switch context into second cluster and apply the output of `istioctl x create-remote-secret` into the first cluster - `istioctl x create-remote-secret | kubectl apply -f - --context <context-name-for-first-cluster>`


# Intra-cluster Load Balancing using ASM - for traffic inside the cluster

    The following steps will demonstrate multi-cluster load balancing within the 2 clusters using the service mesh - Note: This is not the same as Gloabl Load blancing where traffic is coming from a client outside the cluster. This is the scenario wehre services inside the cluster are consuming other services, also deployed on the cluster - and ASM is able to loadbalance such requests b/w the pods on both the clusters.

    In order to do the following, follow the link here : https://cloud.google.com/service-mesh/docs/gke-install-multi-cluster#verify_your_deployment 

    - enable sidecar injection
    - Deploy svc
    - deploy v1 and v2 on cluster1 and 2 respectively
    - deploy sleep containers
    - verify load balancing b/w pods on the 2 clusters using the sleep containers
    - Delete the sample namespace on both the clusters


# Multi-cluster ingress/ Global Loadbalancing setup


Reference: https://docs.google.com/document/d/1VDlGHlYq8uw4JTsq-iMjrLmptQ6gMgtcNHwWFlYbrjg/edit?resourcekey=0-wdodkiXzWXbCrSjnwzLS7A

```bash
export PROJECT=$(gcloud config get-value project)
```

- Create a global static Ip address for the Global LB (HTTP LB)

    ```bash
    gcloud compute addresses create glb-ip --global

    # Note the external IP address created here. If you're not able to see the IP, wait for a couple of minutes before trying
    gcloud compute addresses list --global 

    ```

- Create a DNS entry that links the URL that users will use to access your application, with the GLB-IP.

    ```bash
    cat <<EOF > multicluster/dns-spec.yaml
    swagger: "2.0"
    info:
    description: "Cloud Endpoints DNS"
    title: "Cloud Endpoints DNS"
    version: "1.0.0"
    paths: {}
    host: "frontend.endpoints.${PROJECT}.cloud.goog"
    x-google-endpoints:
    - name: "frontend.endpoints.${PROJECT}.cloud.goog"
    target: "${GCLB_IP}"
    EOF

    gcloud endpoints services deploy multicluster/dns-spec.yaml
    ```


Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress-setup#requirements_for 

- The cluster has to be registered as a member of the fleet of the asm enabled clusters. This can eithe be done from the anthos UI console, or using the command 

    ```bash
    gcloud container hub memberships register gke-eu \
        --project=project-id \
        --gke-uri=uri \
        --service-account-key-file=service-account-key-path
    ```

    Once done, we have to select one of the clusters as the config cluster. This will house the MCI and MCS resources. Only 1 cluster in a fleet can be an active config cluster



    ```bash
    gcloud alpha container hub ingress enable \
    --config-membership=projects/project_id/locations/global/memberships/gke-us   #Command as per documentation

    gcloud alpha container hub ingress enable \
    --config-membership=cluster1  # The actual command I used.
    ```
    > Note: For me the whole path passed as the config-membership param didn't work, so I passed cluster as the value to the config-membership flag.

- Confirm cluster registration as the config cluster

    ```bash
    gcloud alpha container hub ingress describe
    ```

If all is running well here, now to create an actual ingress resource

Reference: https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress 

- Create a namespace for the `zoneprinter` app and register it for istio sidecar injection

    ```bash
    # get the revision label value from the istiod service in the istio-system
    kubectl -n istio-system get pods -l app=istiod

    # output
    NAME                                 READY   STATUS    RESTARTS   AGE
    istiod-asm-1102-3-76b8d69bf9-9mft7   1/1     Running   0          17h
    istiod-asm-1102-3-76b8d69bf9-gvsw9   1/1     Running   0          17h

    # Note the 'asm-1102-3' from the pod name

    # on both the clusters, run these steps and insert the version from the pod name into the label
    kubectl create ns zoneprinter
    kubectl label ns zoneprinter istio-injection- istio.io/rev=asm-1102-3 --overwrite
    ```

- Deploy the zoneprinter application into both the clusters by deploying the `resources/deploy.yaml`
- Deploy the `MultiClusterService` and the `MultiClusterIngress` into the config cluster - `cluster1` 
- Validate the MCI (MultiClusterIngress) deployment status using `kubectl describe mci zone-ingress -n zoneprinter` 
- Retreive the Public IP address of the HTTP LB created using the MCI - `kubectl get mci zone-ingress -o=jsonpath={.status.VIP}`. This VIP will be the same IP reservation we made earlier.
- The site will be available at the FQDN entry we made in the registry earlier. This can be retrieved using `gcloud endpoints services list --format="value(serviceName)"`


Test the services from different regions using your local broswer as well as https://geotargetly.com/geo-browse 


# Multi-cluster service discovery

Create all sorts of routing rules and ingress configurations using - https://cloud.google.com/kubernetes-engine/docs/how-to/multi-cluster-ingress#features 
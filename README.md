# Run build and deployment using Google Cloud Build

1. Create a new project on GCP
2. Run the commands below

```
export PROJECT_ID=your-gcp-project-id-here

gcloud compute networks create default --project=${PROJECT_ID} --subnet-mode=auto --bgp-routing-mode=global 

gcloud compute firewall-rules create default-allow-http --project=${PROJECT_ID} --network=projects/${PROJECT_ID}/global/networks/default --description="Allows connection from any source to any instance on the network using HTTP." --direction=INGRESS --source-ranges=10.128.0.0/9 --action=ALLOW --rules=tcp:80 

gcloud compute firewall-rules create default-allow-icmp --project=${PROJECT_ID} --network=projects/${PROJECT_ID}/global/networks/default --description=Allows\ ICMP\ connections\ from\ any\ source\ to\ any\ instance\ on\ the\ network. --direction=INGRESS --priority=65534 --source-ranges=0.0.0.0/0 --action=ALLOW --rules=icmp

gcloud compute firewall-rules create default-allow-ssh --project=${PROJECT_ID} --network=projects/${PROJECT_ID}/global/networks/default --description=Allows\ TCP\ connections\ from\ any\ source\ to\ any\ instance\ on\ the\ network\ using\ port\ 22. --direction=INGRESS --priority=65534 --source-ranges=0.0.0.0/0 --action=ALLOW --rules=tcp:22

gcloud services enable compute --project=${PROJECT_ID}

export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)"

# Give the Cloud Build service account permission to create compute instances.
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com --role=roles/compute.admin

# Run the Cloud Build script to build the stubbed service and containerize it.
gcloud builds submit --project=${PROJECT_ID}
```

# Prepare the network so that internal VMs with no external IP can access the internet, as necessary to pull docker images.
## Create a Cloud Router
```
gcloud compute routers create nat-router-us-central1 \
    --network default \
    --region us-central1
```
## Set up Cloud NAT on the router
```
gcloud compute routers nats create nat-config \
    --router-region us-central1 \
    --router nat-router-us-central1 \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
```
# Ensure the image is publicly accessible to all users for unauthenticated access, 
This will be applicable when a new VM is spun up below and attempts to pull the image.
```
gsutil iam ch allusers:objectViewer gs://artifacts.${PROJECT_ID}.appspot.com
```
# Create a pair of VMs that run the container image
* the argument --no-address ensure the VM does not have a public IP address 
* the argument --shielded-secure-boot avoids error for violation of constraints/compute.requireShieldedVm
```
gcloud compute instances create-with-container instance-1 --container-image gcr.io/${PROJECT_ID}/stubbed-service --project=${PROJECT_ID} --zone=us-central1-a --no-address --shielded-secure-boot

gcloud compute instances create-with-container instance-2 --container-image gcr.io/${PROJECT_ID}/stubbed-service --project=${PROJECT_ID} --zone=us-central1-a --no-address --shielded-secure-boot
```
# Allow the port 8080 that the container runs on
*NOTICE: This may not be needed. Futher testing required.
```
gcloud compute firewall-rules create allow-8080 --project=${PROJECT_ID} --network=projects/${PROJECT_ID}/global/networks/default --description="Allows connection from any source to any instance on the network using HTTP." --direction=INGRESS --source-ranges=10.128.0.0/9 --action=ALLOW --rules=tcp:8080 
```
# Group the VM instances created so they can be load balanced
``` 
gcloud compute instance-groups unmanaged create stub-servers --project=lb-test-374518 --zone=us-central1-a
gcloud compute instance-groups unmanaged set-named-ports stub-servers --project=lb-test-374518 --zone=us-central1-a \
    --named-ports=stubweb:8080
gcloud compute instance-groups unmanaged add-instances stub-servers --project=lb-test-374518 --zone=us-central1-a \
    --instances=instance-1,instance-2
```

# Allow Google-managed service the access to do health checks for the load balancer we will create
gcloud compute firewall-rules create fw-allow-health-check \
    --network=default \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --rules=tcp:8080

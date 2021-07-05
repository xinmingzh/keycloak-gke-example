# Keycloak Deployment
Deployment of keycloak on GKE. This tutorial mainly followed the guide by piotr szybicki,
adding on details that are missing in the original guide.

[Medium - Deploy Keycloak to Kubernetes cluster on GCP (piotr szybicki)](https://medium.com/12-developer-labors/deploy-keycloak-to-kubernetees-cluster-on-gcp-9a1afa0984f2)

## Setup Network

1. Define env vars
```commandline
export REGION=asia-east1
export ZONE=asia-east1-a
export VPC=auth-network
export SUBNET=keycloak-subnet
export GCP_PROJECT=<YOUR-GCP-PROJECT>
export CLUSTER=keycloak-cluster
export DB_INSTANCE=keycloak-sql
```
2. Create a VPC
```commandline
gcloud compute networks create $VPC \
    --subnet-mode custom \
    --bgp-routing-mode regional
```

3. Reserve address space for the DB
```commandline
gcloud beta compute addresses create mysql-range \
    --global \
    --purpose=VPC_PEERING \
    --addresses=10.1.0.0 \
    --prefix-length=16 \
    --network $VPC
```

4. Add the firewall rule
```commandline
gcloud compute firewall-rules create allow-connect-to-db \
    --direction=ingress \
    --network $VPC \
    --allow tcp:3306,tcp:3307 \
    --source-ranges 10.0.0.0/8 \
    --destination-ranges=10.1.0.0/16
```

5. Create a subnet for the cluster
```commandline
gcloud compute networks subnets create $SUBNET \
    --network $VPC \
    --range=10.2.0.0/16 \
    --region $REGION \
    --enable-flow-logs \
    --enable-private-ip-google-access
```

## Setup Service Account

6. Create service account
```commandline
gcloud iam service-accounts create keycloak-master
```

7. Add key and save it to a file
```commandline
gcloud iam service-accounts keys create key.json \
    --iam-account keycloak-master@<YOUR-GCP-PROJECT>.iam.gserviceaccount.com
```

8. Grant cloud sql viewer to the service account
```commandline
gcloud projects add-iam-policy-binding $GCP_PROJECT \
    --member serviceAccount:keycloak-master@<YOUR-GCP-PROJECT>.iam.gserviceaccount.com \
    --role roles/cloudsql.viewer
```

9. grant cloud sql client to the service account
```commandline
gcloud projects add-iam-policy-binding $GCP_PROJECT \
    --member serviceAccount:keycloak-master@<YOUR-GCP-PROJECT>.iam.gserviceaccount.com \
    --role roles/cloudsql.client
```

## Setup DB
10. Create DB instance

[Medium - Deploy Keycloak to Kubernetes cluster on GCP (piotr szybicki)](https://medium.com/12-developer-labors/deploy-keycloak-to-kubernetees-cluster-on-gcp-9a1afa0984f2)
```commandline
follow guide to create db instance
```

## Setup GKE
11. Create kubernetes cluster
```commandline
gcloud container clusters create $CLUSTER \
    --num-nodes=3 \
    --machine-type=e2-medium \
    --network $VPC \
    --subnetwork $SUBNET \
    --zone $ZONE \
    --enable-autoscaling \
    --max-nodes=3 \
    --min-nodes=3 \
    --enable-ip-alias \
    --scopes=https://www.googleapis.com/auth/sqlservice
```

12. Get credentials
```commandline
gcloud container clusters get-credentials $CLUSTER \
    --zone $ZONE \
    --project $PROJECT
```

13. Create secret with the service account
```commandline
kubectl create secret generic cloudsql-instance-credentials \
    --from-file=service-account.json=key.json
```

14. Create the user on the instance
```commandline
gcloud sql users create keycloak \
    --host=% \
    --instance $DB_INSTANCE \
    --password=keycloak
```

15. Create secret with db credentials
```commandline
kubectl create secret generic cloudsql-db-credentials \
    --from-literal=username=keycloak \
    --from-literal=password=keycloak
```

16. Create secret with keycloak admin credentials
```commandline
kubectl create secret generic admin-credentials \
    --from-literal=username=admin \
    --from-literal=password=admin
```

17. Add view permission for keycloak peer discovery to work
```commandline
kubectl create rolebinding default-viewer \
    --clusterrole=view \
    --serviceaccount=default:default \
    --namespace=default
```

18. Configure variables in keycloak-deployment.yaml

## Deploy Keycloak

19. Create the deployment
```commandline
kubectl apply -f keycloak-deployment.yaml
```

20. Scale the deployment in order to see cluster forming
```commandline
kubectl scale --replicas=3 deployment/keycloak
```

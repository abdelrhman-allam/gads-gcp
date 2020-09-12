## Create the managementnet network
### Create the privatenet network

To create the privatenet network, run the following command:
```bash
gcloud compute networks create privatenet --subnet-mode=custom
```
To create the privatesubnet-us subnet, run the following command:
```bash
gcloud compute networks subnets create privatesubnet-us --network=privatenet --region=us-central1 --range=172.16.0.0/24
```
To create the privatesubnet-eu subnet, run the following command:
```bash
gcloud compute networks subnets create privatesubnet-eu --network=privatenet --region=europe-west1 --range=172.20.0.0/20
```

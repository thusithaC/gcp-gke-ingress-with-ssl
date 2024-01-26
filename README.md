# GCP GKE Internal Load Balancer with SSL


## Reading
https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-internal#gcloud
https://cloud.google.com/kubernetes-engine/docs/how-to/internal-load-balance-ingress#prepare-environment
https://medium.com/@nikhil.nagarajappa/configuring-internal-ingress-gke-9d1c54d589dd
https://cloud.google.com/kubernetes-engine/docs/how-to/internal-load-balance-ingress#https_between_client_and_load_balancer
https://cloud.google.com/kubernetes-engine/docs/how-to/cloud-dns#enable_scope_dns_in_an_existing_cluster
https://stackoverflow.com/questions/55847423/dns-with-gke-internal-load-balancers
https://stackoverflow.com/questions/57898357/ssl-tls-certificate-for-internal-https-load-balancer-gcp?rq=2


## Steps 

- Set up network with manual subnets. 
- Create a Proxy only subnet and setup firewall rule
- Create GKE cluster in the same VPC
- Create a static IP and internal Domain name A record (you might have to create a Zone first)
- Obtain SSL cert for the domain (externally signed)
- Add the cert to GCP as a resource
- Use the linked deployment, service and ingress configuration. 
- Create a VM in the same network and use curl with the signed CA

```shell
gcloud compute networks create lb-network --subnet-mode=custom

gcloud compute networks subnets create proxy-only-subnet \
  --purpose=REGIONAL_MANAGED_PROXY \
  --role=ACTIVE \
  --region=us-west1 \
  --network=lb-network \
  --range=10.129.0.0/23

gcloud compute firewall-rules create allow-proxy-connection \
    --allow=TCP:80 \
    --source-ranges=10.129.0.0/23 \
    --network=lb-network

gcloud container clusters create-auto test-cluster-2 \
    --location=us-west1 \
    --network=lb-network \
    --create-subnetwork name=k8-subnet

gcloud compute addresses create lb-ingress-ip --project=gke-experimentation-412308 --region=us-west1 --subnet=k8-subnet --purpose=GCE_ENDPOINT

gcloud compute addresses describe lb-ingress-ip | grep address

gcloud dns --project=gke-experimentation-412308 managed-zones create lb-zone --description="" --dns-name="lbzone.internal" --visibility="private" --networks="https://www.googleapis.com/compute/v1/projects/gke-experimentation-412308/global/networks/lb-network"

gcloud dns --project=gke-experimentation-412308 record-sets create ingress.lbzone.internal --zone="lb-zone" --type="A" --ttl="60" --rrdatas="10.36.74.7"


gcloud compute ssl-certificates create  lbcert\
    --certificate certs/ingress.lbzone.internal.crt \
    --private-key certs/ingress.lbzone.internal.key \
    --region us-west1

gcloud compute ssh test-vm \
   --zone=us-west1-a

# inside the VM
curl https://ingress.lbzone.internal --cacert rootCA.crt
```


## Self Signed Key generation

https://gist.github.com/fntlnz/cf14feb5a46b2eda428e000157447309
```shell

openssl genrsa -des3 -out rootCA.key 4096
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt

openssl genrsa -out ingress.lbzone.internal.key 2048

openssl req -new -sha256 \
    -key ingress.lbzone.internal.key \
    -subj "/C=AU/ST=VIC/O=TNC/OU=ORG_UNIT/CN=ingress.lbzone.internal" \
    -reqexts SAN \
    -config <(cat /etc/ssl/openssl.cnf <(printf "\n[SAN]\nsubjectAltName=DNS:www.ingress.lbzone.internal")) \
    -out ingress.lbzone.internal.csr

openssl req -in ingress.lbzone.internal.csr -noout -text

openssl x509 -req \
        -extfile <(printf "[v3_req]\nextendedKeyUsage=serverAuth\nsubjectAltName=DNS:ingress.lbzone.internal,DNS:www.ingress.lbzone.internal") \
        -extensions v3_req \
        -days 365 -in ingress.lbzone.internal.csr -CA rootCA.crt -CAkey rootCA.key \
        -CAcreateserial -out ingress.lbzone.internal.crt -sha256

openssl x509 -in ingress.lbzone.internal.crt -text -noout

```
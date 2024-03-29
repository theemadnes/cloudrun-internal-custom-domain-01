# cloudrun-internal-custom-domain-01
setting up an internal Cloud Run service with a custom domain

> note: this assumes that the VPC already has a proxy-only subnet created for it. see [here](https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-internal-serverless) for more info.


### setup

```
export PROJECT=e2m-private-test-01
export REGION=us-central1
export VPC=default
export SUBNET=default
gcloud config set project $PROJECT
```

### create Cloud Run service

```
gcloud beta run deploy whereami \
    --project=$PROJECT \
    --image us-docker.pkg.dev/google-samples/containers/gke/whereami:v1.2.22 \
    --platform=managed --region=$REGION \
    --network $VPC --subnet $SUBNET \
    --ingress internal \
    --vpc-egress=private-ranges-only \
    --no-allow-unauthenticated
```

### create test certificate for the load balancer

```
# generate self-signed certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=whereami.internal.example.com"

# create SSL cert resource
gcloud compute ssl-certificates create whereami-cert \
    --project=$PROJECT \
    --certificate tls.crt \
    --private-key tls.key \
    --region=$REGION
```

### create regional internal ALB using HTTPS & DNS record

```
# create serverless NEG for Cloud Run
gcloud compute network-endpoint-groups create whereami-serverless-neg \
    --project=$PROJECT \
    --region=$REGION \
    --network-endpoint-type=serverless  \
    --cloud-run-service=whereami

# create backend service
gcloud compute backend-services create whereami \
    --project=$PROJECT \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --protocol=HTTP \
    --region=$REGION

# add serverless NEG to the backend service
gcloud compute backend-services add-backend whereami \
    --project=$PROJECT \
    --region=$REGION \
    --network-endpoint-group=whereami-serverless-neg \
    --network-endpoint-group-region=$REGION

# create regional URL map
# note this is just using a default service, so you'll typically want to set up more specific rules
gcloud compute url-maps create whereami-url-map \
    --default-service=whereami \
    --project=$PROJECT \
    --region=$REGION

# create regional target proxy
gcloud compute target-https-proxies create whereami-proxy \
    --url-map=whereami-url-map \
    --ssl-certificates=whereami-cert \
    --project=$PROJECT \
    --region=$REGION

# create regional forwarding rule
gcloud compute forwarding-rules create whereami-rule \
    --project=$PROJECT \
    --region=$REGION \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --network=$VPC \
    --subnet=$SUBNET \
    --target-https-proxy=whereami-proxy \
    --target-https-proxy-region=$REGION \
    --ports=443

export RILB_VIP=$(gcloud compute forwarding-rules describe whereami-rule --project=$PROJECT --region=$REGION --format json | jq ".IPAddress" | tr -d '"')

# create DNS zone (using geotest name, from another example)
gcloud dns managed-zones create geotest \
    --project=$PROJECT \
    --description="" --dns-name="internal.example.com." \
    --visibility="private" \
    --networks="https://www.googleapis.com/compute/v1/projects/${PROJECT}/global/networks/${VPC}"

# create DNS record
gcloud dns record-sets create whereami.internal.example.com. \
    --project=$PROJECT \
    --zone="geotest" --type="A" --ttl="5" \
    --rrdatas=$RILB_VIP

# test calling the service over HTTPS (using -k because the certificate is self-signed)
curl -k -H "Authorization: Bearer $(gcloud auth print-identity-token)" https://${RILB_VIP} -s | jq
{
  "gce_instance_id": "0087599d422a7a84291bb72d88ae624d52cc19ec7bc2196be90c236be7362ecb246d68addde6d832a03bbb909e5b821dd8a2086f0f09a05ebe9925447fba90f925573f",
  "gce_service_account": "603904278888-compute@developer.gserviceaccount.com",
  "host_header": "whereami.internal.example.com",
  "pod_name": "localhost",
  "pod_name_emoji": "🧑🏻‍🌾",
  "project_id": "e2m-private-test-01",
  "timestamp": "2024-01-25T08:49:55",
  "zone": "us-central1-1"
}
```
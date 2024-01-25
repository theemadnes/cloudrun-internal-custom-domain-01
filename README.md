# cloudrun-internal-custom-domain-01
setting up an internal Cloud Run service with a custom domain

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
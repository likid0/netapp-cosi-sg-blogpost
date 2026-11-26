# Part 1 — Provisioning S3 Buckets the Kubernetes Way: COSI with NetApp Trident and StorageGRID

Companion files for Part 1 of the blog post. Everything here was run against a
single-node K3s 1.36 cluster with Trident 26.10.1, against a five-node
StorageGRID 12.2 system.

## Files

| File | Who applies it | What it does |
|------|----------------|--------------|
| `objectstore.yaml` | Administrator | Secret with tenant root creds + `TridentObjectStore` connecting Trident to one StorageGRID tenant |
| `classes.yaml` | Administrator | `BucketClass` and `BucketAccessClass` developers pick from |
| `claim.yaml` | Developer | `BucketClaim` — requests a bucket |
| `access.yaml` | Developer | `BucketAccess` — requests credentials, written to Secret `my-bucket-creds` |
| `pod-writer.yaml` | Developer | Lab test pod: loads the whole Secret with `envFrom`, uploads `hello.txt`, lists the bucket |
| `app-env.yaml` | Developer | How a real app should consume the bucket: maps `COSI_S3_*` onto standard AWS SDK variables |
| `group-policy.json` | Reference | The S3 group policy Trident generates for a ReadWrite `BucketAccess` |

## Order of operations

```bash
# Administrator, once per cluster (after installing Trident with COSI enabled)
kubectl apply -f objectstore.yaml
kubectl -n trident get tridentobjectstore storagegrid-k8sdemo -o jsonpath='{.status.state}'   # -> online
kubectl apply -f classes.yaml

# Developer, in their own namespace
kubectl create namespace cosi-test
kubectl apply -f claim.yaml
kubectl apply -f access.yaml
```

## The sg-s3-ca ConfigMap (needed by app-env.yaml)

`app-env.yaml` verifies the S3 endpoint's TLS certificate through a CA bundle
mounted from a ConfigMap named `sg-s3-ca`. The lab endpoint uses a self-signed
certificate, so create the ConfigMap from the chain the endpoint presents:

```bash
echo | openssl s_client -connect <s3-endpoint>:10443 -showcerts 2>/dev/null \
  | awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' > sg-s3-chain.pem

kubectl -n cosi-test create configmap sg-s3-ca --from-file=ca.pem=sg-s3-chain.pem
```

If your S3 endpoint uses a certificate from a publicly trusted CA, you don't
need any of this: delete the `sg-ca` volume, its `volumeMount` and the
`AWS_CA_BUNDLE` variable from `app-env.yaml`.

> Note on `pod-writer.yaml`: it passes `--no-verify-ssl` instead, which is
> simpler for a throwaway lab pod but not something to copy into real
> workloads. `app-env.yaml` shows the correct approach.

> Note on image pull policy: these pods use `public.ecr.aws/aws-cli/aws-cli:latest`.
> If your nodes cannot reach the registry, pre-pull the image and set
> `imagePullPolicy: IfNotPresent`.

## Running the test pods

```bash
kubectl apply -f pod-writer.yaml
kubectl -n cosi-test logs pod-writer
# upload: ../tmp/hello.txt to s3://trident-bc-<uid>/hello.txt
# <date> <time>         16 hello.txt

kubectl apply -f app-env.yaml
kubectl -n cosi-test logs app-env
# reading s3://trident-bc-<uid>/hello.txt as a plain AWS SDK client
# <date> <time>         16 hello.txt
# hello from COSI
```

## Cleaning up

Trident deletes only **empty** buckets, and `pod-writer` leaves `hello.txt`
behind. Empty the bucket first, while its credentials still exist, then delete
in reverse order.

```bash
# Empty the bucket using the issued credentials (while they still exist)
kubectl apply -f empty-bucket.yaml
kubectl -n cosi-test logs empty-bucket     # delete: s3://trident-bc-<uid>/hello.txt
kubectl -n cosi-test delete pod empty-bucket

# Delete in reverse order
kubectl -n cosi-test delete pod pod-writer app-env --ignore-not-found
kubectl -n cosi-test delete bucketaccess my-bucket-access   # revokes the key, removes the Secret + SG user/group
kubectl -n cosi-test delete bucketclaim my-first-bucket     # deletes the (now empty) bucket
```

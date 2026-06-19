# Runbook: Supply Chain Security Policy

## 1. Purpose

This runbook describes how to validate the container supply chain security flow using GitHub Actions, Trivy, GitHub Container Registry, Cosign, and Sigstore Policy Controller.

The goal is to ensure that only scanned and signed container images are allowed to run in Kubernetes.

## 2. Scope

This runbook applies to the following resources:

| Resource                    | Name                                |
| --------------------------- | ----------------------------------- |
| API image                   | `ghcr.io/ngonguyentruongan/w10-api` |
| Public key                  | `signing/cosign.pub`                |
| Private key                 | `cosign.key`                        |
| CI workflow                 | `.github/workflows/build-push.yml`  |
| Kubernetes namespace        | `demo`                              |
| Policy Controller namespace | `cosign-system`                     |
| ClusterImagePolicy          | `require-signed-images-demo`        |
| Signed test pod             | `signed-w10-api-test`               |
| Unsigned test pod           | `unsigned-nginx-test`               |

## 3. Security Flow

The supply chain security flow is:

```text
Developer pushes code
        |
        v
GitHub Actions builds image
        |
        v
Trivy scans image
        |
        v
If scan passes, image is pushed to GHCR
        |
        v
Cosign signs image
        |
        v
Cosign verifies image signature
        |
        v
Kubernetes admission policy allows signed image
```

## 4. CI Pipeline Requirements

The CI pipeline must perform the following steps:

1. Build Docker image.
2. Scan image with Trivy.
3. Fail CI if HIGH or CRITICAL vulnerabilities are detected.
4. Push image to GitHub Container Registry.
5. Sign image with Cosign.
6. Verify image signature using `signing/cosign.pub`.
7. Update `app-api/rollout.yaml`.

## 5. Required GitHub Secrets

The GitHub repository must contain these Actions secrets:

```text
GHCR_TOKEN
COSIGN_PRIVATE_KEY
COSIGN_PASSWORD
```

Notes:

* `GHCR_TOKEN` must have permission to push packages.
* `COSIGN_PRIVATE_KEY` stores the content of `cosign.key`.
* `COSIGN_PASSWORD` is the password used when generating the Cosign key pair.
* Do not commit `cosign.key`.

## 6. Verify CI Pipeline

Open GitHub Actions and check the workflow:

```text
Actions → Build and Push Image
```

Expected successful steps:

```text
Build Docker image locally
Scan image with Trivy
Push Docker image
Install Cosign
Sign image with Cosign
Verify image signature
```

The workflow should finish with a green check mark.

## 7. Verify Image in Rollout

Check the image used by the API rollout:

```powershell
Get-Content .\app-api\rollout.yaml | Select-String "image:"
```

Expected format:

```text
image: ghcr.io/ngonguyentruongan/w10-api:0.0.1
```

## 8. Verify Image Signature Locally

Check the signed image by digest:

```powershell
cosign verify --key signing/cosign.pub ghcr.io/ngonguyentruongan/w10-api@sha256:<IMAGE_DIGEST>
```

Expected output:

```text
The cosign claims were validated
Existence of the claims in the transparency log was verified offline
The signatures were verified against the specified public key
```

## 9. Policy Controller Validation

Check Policy Controller pod:

```powershell
kubectl get pods -n cosign-system --request-timeout=120s
```

Expected result:

```text
policy-controller-webhook-xxxxx   1/1   Running
```

Check Policy Controller CRDs:

```powershell
kubectl get crd | Select-String "policy.sigstore"
```

Expected result:

```text
clusterimagepolicies.policy.sigstore.dev
trustroots.policy.sigstore.dev
```

Check API resource:

```powershell
kubectl api-resources | Select-String "clusterimagepolicy"
```

Expected result:

```text
clusterimagepolicies
```

## 10. Namespace Opt-in

Policy Controller should enforce policies only in namespaces that are explicitly opted in.

Label namespace `demo`:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true --overwrite --request-timeout=120s
```

Check label:

```powershell
kubectl get namespace demo --show-labels --request-timeout=120s
```

Expected label:

```text
policy.sigstore.dev/include=true
```

## 11. ClusterImagePolicy

Check policy:

```powershell
kubectl get clusterimagepolicy --request-timeout=120s
```

Expected result:

```text
require-signed-images-demo
```

If policy is missing, create it:

```powershell
kubectl create -f .\policy-controller\cluster-image-policy.yaml --validate=false --request-timeout=120s
```

## 12. Test Signed Image

The signed test pod should use image digest format because Policy Controller requires digest-based image references.

Example image:

```text
ghcr.io/ngonguyentruongan/w10-api@sha256:<IMAGE_DIGEST>
```

Run server-side dry run:

```powershell
kubectl apply -f .\signed-w10-api-test.yaml --dry-run=server --request-timeout=120s
```

Expected result:

```text
pod/signed-w10-api-test created (server dry run)
```

This means the signed image was allowed by the admission policy.

## 13. Test Unsigned Image

Run server-side dry run:

```powershell
kubectl apply -f .\unsigned-nginx-test.yaml --dry-run=server --request-timeout=120s
```

Expected result:

```text
admission webhook "policy.sigstore.dev" denied the request
```

Example reject message:

```text
invalid value: nginx:1.25 must be an image digest
```

This means the unsigned or non-compliant image was rejected by the admission policy.

## 14. Troubleshooting

### 14.1 Webhook connection refused

Check pod and endpoint:

```powershell
kubectl get pods -n cosign-system -o wide --request-timeout=120s
kubectl get endpoints -n cosign-system webhook --request-timeout=120s
```

Expected endpoint:

```text
webhook   10.244.x.x:8443
```

### 14.2 Mutating webhook timeout

If `/mutations` timeout occurs, remove the mutating webhook and keep validating webhook:

```powershell
kubectl delete mutatingwebhookconfiguration policy.sigstore.dev --ignore-not-found --request-timeout=120s
```

Check validating webhook still exists:

```powershell
kubectl get validatingwebhookconfiguration | Select-String "policy"
```

### 14.3 Gatekeeper blocks the request first

If Gatekeeper ValidatingAdmissionPolicy blocks testing, remove the broken VAP resources temporarily:

```powershell
kubectl delete validatingadmissionpolicybinding gatekeeper-k8spsphostnetworkingports-disallow-host-network --ignore-not-found --request-timeout=120s
kubectl delete validatingadmissionpolicy gatekeeper-k8spsphostnetworkingports --ignore-not-found --request-timeout=120s
```

### 14.4 Image must be digest

If Policy Controller says the image must be a digest, use:

```text
ghcr.io/ngonguyentruongan/w10-api@sha256:<IMAGE_DIGEST>
```

instead of:

```text
ghcr.io/ngonguyentruongan/w10-api:0.0.1
```

### 14.5 No signatures found

If Policy Controller says `no signatures found`, verify locally:

```powershell
cosign verify --key signing/cosign.pub ghcr.io/ngonguyentruongan/w10-api@sha256:<IMAGE_DIGEST>
```

If local verification succeeds, check:

* GHCR package is public.
* Signature was pushed successfully.
* Policy Controller is healthy.
* Image digest in the pod matches the signed digest.

## 15. Evidence to Capture

Capture screenshots of:

1. GitHub Actions pass.
2. Trivy scan pass.
3. Cosign sign and verify pass.
4. `cosign verify` local pass.
5. Policy Controller pod `1/1 Running`.
6. `clusterimagepolicies.policy.sigstore.dev` exists.
7. Namespace `demo` has label `policy.sigstore.dev/include=true`.
8. ClusterImagePolicy exists.
9. Signed image dry-run is allowed.
10. Unsigned image dry-run is rejected.

## 16. Final Result

Lab 2.2 was completed successfully. The CI pipeline scanned, pushed, signed, and verified the API image. Sigstore Policy Controller enforced image admission policy in Kubernetes. The signed image was allowed, and the unsigned image was rejected.

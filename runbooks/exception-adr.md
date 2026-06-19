# ADR: Lab Exceptions and Local Environment Notes

## 1. Status

Accepted for lab environment.

## 2. Context

Lab 2 was executed on a local Minikube cluster. The lab enabled multiple security controllers, including:

* External Secrets Operator
* Gatekeeper
* Sigstore Policy Controller
* ArgoCD

Because the environment was local and resource-limited, some temporary operational issues occurred, including Kubernetes API timeouts, webhook readiness issues, and admission policy conflicts.

The main purpose of this ADR is to document the exceptions and manual steps used to complete the lab safely.

## 3. Decisions

### 3.1 AWS credentials are created manually

AWS credentials were created manually as a Kubernetes Secret for the lab environment.

Decision:

```text
Do not commit AWS credentials to Git.
```

Reason:

The lab uses static AWS credentials for local testing. In a production EKS environment, IRSA should be used instead.

### 3.2 Cosign private key is not committed

The file `cosign.key` is kept local and is not committed to the repository.

Decision:

```text
Commit signing/cosign.pub only.
Do not commit cosign.key.
```

Reason:

`cosign.key` is a private signing key. Committing it would compromise the image signing process.

### 3.3 Public key is committed

The file `signing/cosign.pub` is committed.

Decision:

```text
Commit signing/cosign.pub.
```

Reason:

The public key is required by CI verification and Kubernetes admission policy.

### 3.4 Temporary troubleshooting files are not final deliverables

Some temporary files were created during troubleshooting, such as:

```text
policy-controller-disable-probe.json
policy-controller-probe-patch.json
patch-api-replicas.json
gatekeeper-status.yaml
```

Decision:

```text
Do not include temporary troubleshooting files as required final deliverables.
```

Reason:

They were only used to debug the local Minikube environment and are not part of the intended GitOps configuration.

### 3.5 Gatekeeper VAP conflict was temporarily removed

During Sigstore testing, a Gatekeeper ValidatingAdmissionPolicy blocked requests because it referenced a missing or unavailable parameter kind.

Example issue:

```text
ValidatingAdmissionPolicy 'gatekeeper-k8spsphostnetworkingports'
failed to find resource referenced by paramKind: K8sPSPHostNetworkingPorts
```

Decision:

```text
Temporarily remove the broken Gatekeeper ValidatingAdmissionPolicy and binding during Sigstore testing.
```

Reason:

The broken Gatekeeper VAP prevented testing of Sigstore Policy Controller. Removing it allowed the Sigstore admission policy to be validated.

### 3.6 Signed image testing uses digest format

Policy Controller required digest-based image references during admission testing.

Decision:

```text
Use image digest format for signed image test.
```

Example:

```text
ghcr.io/ngonguyentruongan/w10-api@sha256:<IMAGE_DIGEST>
```

Reason:

Using a digest ensures the exact signed artifact is tested.

### 3.7 Mutating webhook was disabled if it blocked testing

During testing, the Sigstore mutating webhook sometimes timed out.

Decision:

```text
Disable mutating webhook if it blocks testing, while keeping validating webhook enabled.
```

Reason:

The validating webhook is responsible for enforcing allow or deny decisions. The lab goal is to demonstrate admission enforcement.

## 4. Consequences

The accepted exceptions allowed the lab to be completed in a local Minikube environment while preserving the important security goals:

* No private key was committed.
* No AWS credentials were committed.
* The image was scanned with Trivy.
* The image was signed with Cosign.
* The image signature was verified.
* The signed image was allowed.
* The unsigned image was rejected.

## 5. Security Considerations

The following items must not be committed to Git:

```text
cosign.key
AWS access key
AWS secret access key
GHCR_TOKEN
COSIGN_PRIVATE_KEY
COSIGN_PASSWORD
.env files containing secrets
```

The following items are safe to commit:

```text
signing/cosign.pub
eso/ manifests
policy-controller/ manifests
.github/workflows/
runbooks/
Evidence.md
```

## 6. Final Validation

The final validation results were:

```text
ESO secret rotation: successful
Trivy scan: successful
Cosign sign and verify: successful
Policy Controller installed: successful
Signed image admission: allowed
Unsigned image admission: rejected
```

## 7. Final Conclusion

The lab was completed successfully. The local Minikube environment required temporary operational exceptions, but the main security requirements were achieved. Secret rotation, image scanning, image signing, signature verification, and admission policy enforcement were all demonstrated.

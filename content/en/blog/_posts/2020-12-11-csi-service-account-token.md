---
layout: blog
title: "Pod Impersonation and Short-lived Volumes in CSI Drivers"
date: 2020-12-11
---

**Author:** Shihang Zhang (Google)

Typically, when a CSI driver mounts credentials such as secrets and
certificates, it has to authenticate to storage providers to be trusted to
access the credentials. However, those credentials are access controlled based
on the pods identities rather than the CSI driver's identity. CSI drivers need
some way to retrieve pod's service account token.

There are two suboptimal approaches:

1.  Grant CSI drivers the permission to use TokenRequest API
2.  Read tokens directly from host filesystem

Both of them exhibits the following drawbacks:

- Violates the principle of least privilege
- Every CSI driver needs to re-implement

Kubernetes 1.20 introduces an alpha feature `CSIServiceAccountToken` to improve
the security posture, which enables CSI drivers to receive pod's
[bound service account token](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/1205-bound-service-account-tokens/README.md).

This feature also provides a knob to re-publish volumes so that short-lived
volumes can be refreshed.

## Pod Impersonation

### Using GCP APIs

With
[Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity),
a Kubernetes service account can authenticates as a Google service account when
accessing Google Cloud APIs. If a CSI driver needs to access GCP APIs on behalf
of the pods that it is mounting volumes for, it can use the pod's service
account token to
[exchange for GCP tokens](https://cloud.google.com/iam/docs/reference/sts/rest).
The pod's service account token is plumbed through the volume context in
`NodePublishVolume` rpc calls when the feature `CSIServiceAccountToken` is
enabled. For example, accessing
[Google Secret Manager](https://cloud.google.com/secret-manager/) via
[secret store CSI driver](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp).

### Using Vault

If users configure
[Kubernetes as an auth method](https://www.vaultproject.io/docs/auth/kubernetes),
Vault uses the `TokenReview` API to validate the Kubernetes service account
token. For CSI drivers using Vault as resources provider, they need to present
the pod's service account to Vault. For example,
[secrets store CSI driver](https://github.com/hashicorp/secrets-store-csi-driver-provider-vault)
and [cert manager CSI driver](https://github.com/jetstack/cert-manager-csi).

## Short-lived Volumes

To keep short-lived volumes such as certificates effective, users can specify
`RequiresRepublish=true` in `CSIDriver` object to have Kubelet periodically
calling `NodePublishVolume` on mounted volumes so that CSI drivers can make sure
the content are up-to-date.

## Next steps

This feature is an alpha feature and projected to move to beta in 1.21. See more
in the following KEP and CSI documentation:

- [KEP-1855: Service Account Token for CSI Driver](https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/1855-csi-driver-service-account-token/README.md)
- [Token Requests](https://kubernetes-csi.github.io/docs/token-requests.html)

Your feedback is very welcomed. SIG-Storage
[meets regularly](https://github.com/kubernetes/community/tree/master/sig-storage#meetings)
and can be reached via
[Slack and a mailing list](https://github.com/kubernetes/community/tree/master/sig-storage#contact).

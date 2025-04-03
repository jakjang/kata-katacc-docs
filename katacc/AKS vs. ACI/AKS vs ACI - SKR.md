*This document aims to illustrate the differences between the AKS and ACI Confidential Containers features. This exercise should help to hightlight
differences in the Kata vs. ACI implementation (colloquially the "passport model"). For this first pass, only the SKR process is highlighted.*

*The process captured here are based heavily off the examples given [here](https://github.com/microsoft/confidential-sidecar-containers/tree/main/examples/skr)
rather than the process documented in the respective learn.microsoft pages*

| **Version** | **Date** | **Changes**|
| --- | --- | --- |
| 0.1 | April 3rd, 2025 | Porting to GitHub and MD |

# ACI Process
The process, broadly, looks like:
1. Grab your mHSM instance. You can use AKV or BYO. 
1. Grab an attestation endpoint (either make your own or use the [generic ones](https://github.com/microsoft/confidential-sidecar-containers/tree/main/examples/skr/aks)
1. Generate User Managed Identity.
*basically diverges from AKS at this point*
1. Update the image registry creds in your ARM template to access a CR.
1. Get an AAD token.
1. Fill in key information in the `importkeyconfig.json`.
1. Generate security policy.
   - *This becomes similar again. Only for ACI, it's done on your ARM template, for AKS, its for your pod YAML*.
1. Import Keys into your mHSM instance.
1. Deploy your ACI template.

# AKS Process
The process, broadly, looks like:
1. Grab your mHSM instance. You can use AKV or BYO. 
1. Grab an attestation endpoint (either make your own or use the [generic ones](https://github.com/microsoft/confidential-sidecar-containers/tree/main/examples/skr/aks)
1. Generate a key pair.
   - Customers will be interacting with the key for wrapping their secret.
1. Build a Secure Key Release (SKR) binary.
1. Create your Federated Cred Identity.
1. Build required container images
   - These depend on how you plan to unwrap your secrets. Needs a little more digging into for possibilities.
   - Right now, broadly aware of using either Kafka streaks or using SKR containers as a gRPC service.
1. Build/run your pod manifest.
   - In every example I've seen, there's a secure key release sidecar that is used for proper key release.
   - Looks to be unavoidable. the open-soruce conf sidecar container is used as the attestation client.



### Introduction

A simple demo showing how to use cosign and kyverno to enforce image signing. It is assumed you have kyverno already installed in the target cluster.

### Steps

1. Generate a cosign keypair to use for signing images:

```
$ cosign generate-key-pair
Enter password for private key: 
Enter password for private key again: 
Private key written to cosign.key
Public key written to cosign.pub
```

2. Update the test image to point to a registry/repository you have access to

```
podman pull quay.io/gnunn/simple-httpd:latest
podman tag quay.io/gnunn/simple-httpd:latest <your-registry>/<your-repository>/simple-httpd:latest
podman push <your-registry>/<your-repository>/simple-httpd:latest
```

3. Update the kyverno-policy with your public key that you generated in step 1 and the image reference from step 2

4. Create an OpenShift project (or k8s namespace)

```oc new-project cosign-demo```

5. Deploy the kyverno policy

```oc apply -f kyverno-policy.yaml```

6. Update the image in `manifests/deployment.yaml` to reference your image instead of the one there

7. Try to deploy the application

```oc apply -k ./manifests```

8. You should see that the deployment failed because the image could not be verified:

```
$ oc apply -k ./manifests
service/practice-test created
route.route.openshift.io/practice-test created
Error from server: error when creating "./manifests": admission webhook "mutate.kyverno.svc-fail" denied the request: 

resource Deployment/cosign-demo/practice-test was blocked due to the following policies

verify-image:
  verify-image: |
    failed to verify signature for quay.io/gnunn/simple-httpd:latest: .attestors[0].entries[0].keys: no matching signatures:
```


9. Sign the image, note to do this you need to have your credentials in `~/.docker/docker` in order for cosign to be able to update the image

```
$ cosign sign --key cosign.key <your-registry>/<your-repository>/simple-httpd:latest
Enter password for private key: 
Warning: Tag used in reference to identify the image. Consider supplying the digest for immutability.
Pushing signature to: quay.io/gnunn/simple-httpd
```

10. Verify using cosign that the image has been signed:

```cosign verify --key cosign.pub <your-registry>/<your-repository>/simple-httpd:latest```

11. Verify in your registry that the image is signed, in quay a little shield icon will appear next to the tag.

12. Apply the manifests again:

```oc apply -k ./manifests```
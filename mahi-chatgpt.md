To securely store and manage Robusta's secret sink credentials (e.g., API keys, configuration) using the Sealed Secrets method in your existing AKS cluster, you can follow this comprehensive step-by-step guide.

Step 1: Install Sealed Secrets Controller in AKS
Add the Sealed Secrets Helm Repository:

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
```
Install the Sealed Secrets Controller:

```
helm install sealed-secrets sealed-secrets/sealed-secrets --namespace kube-system
```
Verify Installation: Check if the controller is running:

````
kubectl get pods -n kube-system | grep sealed-secrets
```
The output should show the sealed-secrets-controller pod running.

Step 2: Create and Encrypt Robusta Credentials
Prepare a Secret File: Create a Kubernetes Secret manifest (robusta-secret.yaml) to hold the sink credentials. Replace <base64_encoded_value> with actual Base64-encoded credentials:

```
apiVersion: v1
kind: Secret
metadata:
  name: robusta-secret
  namespace robusta
type: Opaque
data:
  SLACK_API_KEY: <base64_encoded_slack_api_key>
  CONFIG_JSON: <base64_encoded_config_json>
```
To encode values to Base64, use:

```
echo -n 'your_value' | base64
```
Encrypt the Secret Using kubeseal: Use the public key of the Sealed Secrets controller to encrypt the secret:

```
kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system --format=yaml < robusta-secret.yaml > robusta-sealed-secret.yaml
```
The output file robusta-sealed-secret.yaml contains an encrypted version of the secret.

Verify the Encrypted Secret: Open and inspect robusta-sealed-secret.yaml to confirm it’s encrypted.

Step 3: Store Sealed Secret in Version Control
Add robusta-sealed-secret.yaml to your Git repository:
It is safe to store sealed secrets in your version control system because they cannot be decrypted without the private key stored in the Sealed Secrets controller.
Step 4: Apply the Sealed Secret to AKS
Deploy the Sealed Secret: Apply the sealed secret to your AKS cluster:

```
kubectl apply -f robusta-sealed-secret.yaml
```
Verify the Kubernetes Secret: The Sealed Secrets controller automatically decrypts the sealed secret and creates a regular Kubernetes Secret. Check for its existence:

```
kubectl get secret robusta-secret -n robusta
```
You should see the robusta-secret in the namespace robusta.

Step 5: Reference the Secret in Robusta Values
Modify the values.yaml for Robusta: Update the values.yaml file to use the existing secret:

```
sinks:
  - name: slack
    params:
      api_key:
        valueFrom:
          secretKeyRef:
            name: robusta-secret
            key: SLACK_API_KEY
config:
  existingSecret:
    name: robusta-secret
    key: CONFIG_JSON
```
The valueFrom.secretKeyRef ensures the sink credentials (e.g., SLACK_API_KEY) are securely fetched from the Kubernetes secret.
Similarly, the configuration (CONFIG_JSON) is fetched from the same secret.
Deploy Robusta with Helm: Install or upgrade Robusta using Helm, passing the updated values.yaml file:

```
helm upgrade --install robusta robusta/robusta -n robusta -f values.yaml
```
How Sealed Secrets Work
Encryption:

kubeseal encrypts the sensitive data in a Kubernetes Secret using the public key of the Sealed Secrets controller.
The resulting sealed secret is a custom Kubernetes resource (SealedSecret) that is safe to store in version control.
Decryption:

When the SealedSecret is applied to the cluster, the Sealed Secrets controller uses its private key to decrypt the secret and create a standard Kubernetes Secret.
Security:

Only the cluster with the corresponding private key (managed by the Sealed Secrets controller) can decrypt the secret.
Secrets remain encrypted and secure even if the sealed secret file is leaked.
Step 6: Automate with GitLab CI/CD
Store Public Key in GitLab:

Export the public key of the Sealed Secrets controller:
```
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml | grep tls.crt | awk '{print $2}' | base64 --decode > sealed-secrets-public.pem
```
Add the public key content as a GitLab variable (e.g., SEALED_SECRETS_PUBLIC_KEY).
GitLab CI/CD Pipeline: Add the following to .gitlab-ci.yml to automate the encryption and deployment:

```
stages:
  - encrypt-secrets
  - deploy

encrypt-secrets:
  stage: encrypt-secrets
  image: bitnami/kubeseal:latest
  script:
    - echo "$SEALED_SECRETS_PUBLIC_KEY" > sealed-secrets-public.pem
    - echo "apiVersion: v1
      kind: Secret
      metadata:
        name: robusta-secret
        namespace: robusta
      type: Opaque
      data:
        SLACK_API_KEY: $(echo -n 'your_slack_key' | base64)
        CONFIG_JSON: $(echo -n 'your_config_json' | base64)" > robusta-secret.yaml
    - kubeseal --cert sealed-secrets-public.pem --format=yaml < robusta-secret.yaml > robusta-sealed-secret.yaml
    - cat robusta-sealed-secret.yaml
    - mv robusta-sealed-secret.yaml /artifacts/

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - echo "$KUBE_CONFIG" > kubeconfig
    - export KUBECONFIG=kubeconfig
    - kubectl apply -f /artifacts/robusta-sealed-secret.yaml
```
This method ensures that your Robusta sink credentials and configuration are securely stored, managed, and deployed in your existing AKS cluster, leveraging GitLab CI/CD for automation.

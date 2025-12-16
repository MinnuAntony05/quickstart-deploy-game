# Complete Installation Steps - NGINX Ingress Controller Setup

## Prerequisites

```bash
# Connect to your GKE cluster
gcloud container clusters get-credentials <CLUSTER_NAME> --region <REGION>

# Verify cluster connection
kubectl cluster-info
kubectl get nodes
```

## Step 1: Install NGINX Ingress Controller using Helm

```bash
# Add NGINX Ingress Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller (GKE Autopilot compatible)
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Wait for installation to complete
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Verify ingress controller is running
kubectl get pods -n ingress-nginx
kubectl get deploy -n ingress-nginx
```

## Step 2: Get the External IP Address

```bash
# Wait for LoadBalancer to assign external IP (may take 2-3 minutes)
kubectl get svc -n ingress-nginx

# Watch until EXTERNAL-IP appears
kubectl get svc -n ingress-nginx ingress-nginx-controller --watch

# Once you see the EXTERNAL-IP (e.g., 34.31.31.179), press Ctrl+C
# Save the IP address
export INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Your Ingress External IP: $INGRESS_IP"

# Verify IngressClass is created
kubectl get ingressclass
# Should show: nginx   k8s.io/ingress-nginx
```

## Step 3: Configure DNS

**Update your DNS provider to point your domain to the Ingress IP:**

```
Type: A Record
Name: games-ai-gcp (or @)
Value: <INGRESS_IP from Step 2>
TTL: 300 (5 minutes)
```

**Full domain:** `games-ai-gcp.ustpace.com` → `<INGRESS_IP>`

**Verify DNS propagation:**
```bash
nslookup games-ai-gcp.ustpace.com
dig games-ai-gcp.ustpace.com
```

## Step 4: Install cert-manager

```bash
# Add Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager with CRDs (GKE Autopilot compatible)
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.2 \
  --set crds.enabled=true

# Verify cert-manager installation
kubectl get pods -n cert-manager

# Wait for all cert-manager pods to be ready
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --all \
  --timeout=120s

# Verify cert-manager components
kubectl get deploy -n cert-manager
# Should show: cert-manager, cert-manager-cainjector, cert-manager-webhook
```

## Step 5: Update Email in Certificate Configuration

**IMPORTANT: Edit the certificate file and add your email address**

```bash
# Open the certificate file
nano k8s/04-certificate.yaml

# Uncomment and update this line in the file:
# Change: #email: your-email@ustpace.com
# To:     email: minnu.antony@ustpace.com
```

Or use this command:
```bash
sed -i 's/#email: your-email@ustpace.com/email: minnu.antony@ustpace.com/' k8s/04-certificate.yaml
```

## Step 6: Deploy Application to Kubernetes

```bash
# Navigate to project directory
cd /home/ust-minnu-antony/PACE-Learning/quickstart-deploy-game

# Apply manifests in order
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/01-storage.yaml
kubectl apply -f k8s/02-deployment.yaml
kubectl apply -f k8s/03-service.yaml

# Apply certificate configuration (ClusterIssuers)
kubectl apply -f k8s/04-certificate.yaml

# Verify ClusterIssuers are ready
kubectl get clusterissuer
# Should show: letsencrypt-prod (Ready=True), letsencrypt-staging (Ready=True)

# Apply ingress configuration
kubectl apply -f k8s/05-ingress.yaml

# Verify resources
kubectl get all -n game-2048
kubectl get ingress -n game-2048
```

## Step 7: Verify Deployment

```bash
# Check all resources in namespace
kubectl get all -n game-2048

# Expected output:
# - deployment.apps/deployment-2048 (2/2 READY)
# - service/service-2048 (ClusterIP)
# - 2 pods running

# Check pods status
kubectl get pods -n game-2048

# Check service
kubectl get svc -n game-2048

# Check ingress details
kubectl get ingress -n game-2048
kubectl describe ingress ingress-2048 -n game-2048

# Verify ingress has the LoadBalancer IP assigned
# Look for: LoadBalancer Ingress: 34.31.31.179

# Check ingress configuration
kubectl get ingress -n game-2048 -o yaml
```

## Step 8: Monitor Certificate Issuance

**Note:** cert-manager automatically creates the Certificate resource when it detects the Ingress with the `cert-manager.io/cluster-issuer` annotation.

```bash
# Check if certificate was automatically created
kubectl get certificate -n game-2048

# Watch certificate until it shows Ready=True (may take 3-10 minutes)
kubectl get certificate -n game-2048 --watch

# Check certificate details
kubectl describe certificate game-2048-tls-secret -n game-2048

# Check the TLS secret was created
kubectl get secret game-2048-tls-secret -n game-2048

# Verify secret type is kubernetes.io/tls
kubectl get secret game-2048-tls-secret -n game-2048 -o yaml | grep "type:"

# If there are issues, check certificate requests
kubectl get certificaterequest -n game-2048

# Check challenges (HTTP-01 validation)
kubectl get challenges -n game-2048

# Check cert-manager logs if needed
kubectl logs -n cert-manager -l app=cert-manager --tail=50
```

## Step 9: Test Application Access

```bash
# Test HTTP (should redirect to HTTPS due to force-ssl-redirect annotation)
curl -I http://games-ai-gcp.ustpace.com
# Expected: HTTP/1.1 308 Permanent Redirect

# Test HTTPS (should return 200 OK)
curl -I https://games-ai-gcp.ustpace.com
# Expected: HTTP/2 200

# Check certificate details and issuer
curl -vI https://games-ai-gcp.ustpace.com 2>&1 | grep -E '(subject|issuer|SSL certificate verify)'

# Verify certificate is from Let's Encrypt
# Should show: issuer: C=US; O=Let's Encrypt; CN=R12

# Alternative: Use openssl to check certificate
echo | openssl s_client -servername games-ai-gcp.ustpace.com -connect games-ai-gcp.ustpace.com:443 2>/dev/null | openssl x509 -noout -issuer -dates
```

## Step 10: Access in Browser

Open your browser and visit:
```
https://games-ai-gcp.ustpace.com
```

You should see:
- ✅ Valid SSL certificate (Let's Encrypt)
- ✅ 2048 game application running
- ✅ No security warnings

---

## Additional Verification Commands

### Check Ingress NGINX Controller
```bash
# Check controller deployment
kubectl get deploy -n ingress-nginx

# Check controller pods
kubectl get pods -n ingress-nginx

# Check controller service and external IP
kubectl get svc -n ingress-nginx

# Check IngressClass
kubectl get ingressclass
```

### Check cert-manager
```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check ClusterIssuers
kubectl get clusterissuer

# Check all certificates in cluster
kubectl get certificate --all-namespaces
```

### Debug Certificate Issues
```bash
# Describe certificate for events
kubectl describe certificate game-2048-tls-secret -n game-2048

# Check certificate details including expiry
kubectl get certificate game-2048-tls-secret -n game-2048 -o yaml

# Check secret annotations
kubectl get secret game-2048-tls-secret -n game-2048 -o yaml | grep -A 10 "annotations:"
```

---

## Troubleshooting

### Issue: External IP shows <pending>
```bash
# Wait longer (up to 5 minutes)
kubectl get svc -n ingress-nginx ingress-nginx-controller --watch
```

### Issue: Certificate not Ready
```bash
# Check certificate status and events
kubectl get certificate -n game-2048
kubectl describe certificate game-2048-tls-secret -n game-2048

# Check if DNS is pointing correctly to the LoadBalancer IP
nslookup games-ai-gcp.ustpace.com
dig games-ai-gcp.ustpace.com

# Verify DNS points to the ingress external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Check certificate requests
kubectl get certificaterequest -n game-2048
kubectl describe certificaterequest -n game-2048

# Check HTTP-01 challenges
kubectl get challenges -n game-2048
kubectl describe challenges -n game-2048

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=100 -f

# If certificate is stuck, delete and let it recreate
kubectl delete certificate game-2048-tls-secret -n game-2048
kubectl delete secret game-2048-tls-secret -n game-2048
# Wait 2-3 minutes, cert-manager will recreate automatically
```

### Issue: Pods not running
```bash
# Check pod status
kubectl get pods -n game-2048

# Describe pod
kubectl describe pod <pod-name> -n game-2048

# Check logs
kubectl logs <pod-name> -n game-2048

# Check events
kubectl get events -n game-2048 --sort-by='.lastTimestamp'
```

### Issue: 404 or 503 errors
```bash
# Check ingress
kubectl describe ingress ingress-2048 -n game-2048

# Check service endpoints
kubectl get endpoints service-2048 -n game-2048

# Check NGINX logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100
```

---

## Common Commands

### View Application Logs
```bash
kubectl logs -n game-2048 -l app.kubernetes.io/name=app-2048 --tail=100 -f
```

### Scale Deployment
```bash
kubectl scale deployment deployment-2048 -n game-2048 --replicas=3
```

### Update Application Image
```bash
kubectl set image deployment/deployment-2048 \
  app-2048=public.ecr.aws/l6m2t8p7/docker-2048:latest \
  -n game-2048

kubectl rollout status deployment/deployment-2048 -n game-2048
```

### Delete Everything
```bash
# Delete application
kubectl delete -f k8s/

# Delete NGINX Ingress
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx

# Delete cert-manager
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
```

---

## Summary

✅ **Step 1:** Install NGINX Ingress Controller via Helm  
✅ **Step 2:** Get LoadBalancer External IP and verify IngressClass  
✅ **Step 3:** Configure DNS A record pointing to external IP  
✅ **Step 4:** Install cert-manager v1.19.2 with CRDs  
✅ **Step 5:** Update email in certificate ClusterIssuer config  
✅ **Step 6:** Deploy application manifests (namespace, storage, deployment, service, certificate, ingress)  
✅ **Step 7:** Verify deployment status and ingress configuration  
✅ **Step 8:** Monitor automatic certificate provisioning by cert-manager  
✅ **Step 9:** Test HTTP redirect and HTTPS access with curl  
✅ **Step 10:** Access application in browser with valid SSL  

**Components Installed:**
- **Ingress NGINX Controller** v1.14.1 (in `ingress-nginx` namespace)
- **cert-manager** v1.19.2 (in `cert-manager` namespace)
- **ClusterIssuers**: letsencrypt-prod, letsencrypt-staging
- **Application**: 2048 game (in `game-2048` namespace)

**Automatic Certificate Management:**
- cert-manager watches Ingress resources with `cert-manager.io/cluster-issuer` annotation
- Automatically creates Certificate resources
- Handles ACME HTTP-01 challenge via nginx ingress
- Obtains Let's Encrypt TLS certificate
- Stores certificate in Secret (type: kubernetes.io/tls)
- Auto-renews 30 days before expiry

**Total time:** ~15-20 minutes (including DNS propagation and certificate issuance)

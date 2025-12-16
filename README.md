# Game 2048 - Enterprise GKE Autopilot Deployment

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.27+-blue.svg)](https://kubernetes.io/)
[![GKE Autopilot](https://img.shields.io/badge/GKE-Autopilot-4285F4.svg)](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)
[![NGINX Ingress](https://img.shields.io/badge/NGINX-Ingress-009639.svg)](https://kubernetes.github.io/ingress-nginx/)
[![cert-manager](https://img.shields.io/badge/cert--manager-1.14-326CE5.svg)](https://cert-manager.io/)

Production-ready deployment of the 2048 game on Google Kubernetes Engine (GKE) Autopilot with NGINX Ingress Controller, automated TLS certificate management, and enterprise-grade security and scalability features.

## 🚀 Features

### Infrastructure
- **NGINX Ingress Controller** - High-performance L7 load balancing and routing
- **cert-manager** - Automated TLS certificate management with Let's Encrypt
- **GKE Autopilot** - Fully managed, optimized Kubernetes clusters

### High Availability
- Multi-replica deployment (3 pods) across availability zones
- Pod Disruption Budget (PDB) - maintains minimum 2 pods during disruptions
- Regional persistent disks - storage replicated across zones
- Anti-affinity rules - spreads pods across nodes

### Auto-scaling
- Horizontal Pod Autoscaler (HPA) - scales 3-20 pods based on CPU/Memory
- NGINX Ingress autoscaling - scales 2-10 controller replicas
- Smart scaling policies - fast scale-up, gradual scale-down

### Security
- Network Policies - restricts ingress/egress traffic
- Pod Security Standards - non-root user, dropped capabilities
- Security headers - X-Frame-Options, XSS-Protection, CSP
- Rate limiting - 100 RPS per IP, 50 concurrent connections
- TLS/HTTPS enforcement - automated certificate rotation
- Resource quotas and limits

### Reliability
- Health checks - liveness, readiness, and startup probes
- Rolling updates - zero-downtime deployments
- Session affinity - consistent user experience
- Graceful shutdown - 30-second termination period

## 📋 Prerequisites

- GKE Autopilot cluster
- kubectl (v1.24+)
- Helm 3.x
- gcloud CLI
- Domain name (e.g., games-ai-gcp.ustpace.com)

## 🛠️ Quick Start

See [QUICK-START.md](QUICK-START.md) for condensed deployment commands.

## 📖 Complete Documentation

- **[NGINX-INGRESS-SETUP.md](NGINX-INGRESS-SETUP.md)** - Detailed NGINX Ingress Controller installation guide
- **[DEPLOYMENT-GUIDE.md](DEPLOYMENT-GUIDE.md)** - Comprehensive enterprise deployment guide
- **[QUICK-START.md](QUICK-START.md)** - Quick reference commands

## 🗂️ Project Structure

```
.
├── k8s/
│   ├── 00-namespace.yaml           # Namespace definition
│   ├── 01-storage.yaml             # StorageClass and PVC
│   ├── 02-deployment.yaml          # Application deployment with enterprise configs
│   ├── 03-service.yaml             # ClusterIP service
│   ├── 04-certificate.yaml         # cert-manager ClusterIssuers and Certificate
│   ├── 05-ingress.yaml             # NGINX Ingress with security annotations
│   ├── 06-hpa.yaml                 # Horizontal Pod Autoscaler
│   ├── 07-pdb.yaml                 # Pod Disruption Budget
│   ├── 08-resource-limits.yaml     # ResourceQuota and LimitRange
│   ├── 09-networkpolicy.yaml       # Network security policies
│   └── 10-monitoring.yaml          # ServiceMonitor (optional)
├── NGINX-INGRESS-SETUP.md          # NGINX setup instructions
├── DEPLOYMENT-GUIDE.md             # Complete deployment guide
├── QUICK-START.md                  # Quick reference
└── README.md                       # This file
```

## 🚀 Deployment Steps

### 1. Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.autoscaling.enabled=true \
  --set controller.autoscaling.minReplicas=2 \
  --set controller.autoscaling.maxReplicas=10
```

### 2. Configure DNS

```bash
# Get external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Point DNS A record: games-ai-gcp.ustpace.com -> <EXTERNAL_IP>
```

### 3. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.0
```

### 4. Update Configuration

Edit `k8s/04-certificate.yaml` and replace:
```yaml
email: your-email@ustpace.com  # Your actual email
```

### 5. Deploy Application

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/08-resource-limits.yaml
kubectl apply -f k8s/01-storage.yaml
kubectl apply -f k8s/02-deployment.yaml
kubectl apply -f k8s/03-service.yaml
kubectl apply -f k8s/04-certificate.yaml
kubectl apply -f k8s/05-ingress.yaml
kubectl apply -f k8s/06-hpa.yaml
kubectl apply -f k8s/07-pdb.yaml
kubectl apply -f k8s/09-networkpolicy.yaml
```

### 6. Verify Deployment

```bash
# Check all resources
kubectl get all,ingress,certificate,hpa,pdb -n game-2048

# Wait for certificate (5-10 minutes)
kubectl get certificate -n game-2048 -w

# Access application
https://games-ai-gcp.ustpace.com
```

## 📊 Monitoring

```bash
# View pods and resources
kubectl get all -n game-2048

# Check HPA status
kubectl get hpa -n game-2048

# View logs
kubectl logs -n game-2048 -l app.kubernetes.io/name=app-2048 --tail=100

# Check resource usage
kubectl top pods -n game-2048
```

## 🔧 Configuration

### Resource Limits

- **CPU Request**: 250m per pod
- **Memory Request**: 256Mi per pod
- **CPU Limit**: 500m per pod
- **Memory Limit**: 512Mi per pod

### Autoscaling

- **Min Replicas**: 3
- **Max Replicas**: 20
- **CPU Target**: 70%
- **Memory Target**: 80%

### Security

- **Rate Limiting**: 100 requests/second per IP
- **Connection Limit**: 50 concurrent connections per IP
- **TLS**: Enforced HTTPS with automatic certificate renewal
- **Security Headers**: X-Frame-Options, CSP, XSS-Protection

## 🔍 Troubleshooting

### Certificate Issues

```bash
# Check certificate status
kubectl describe certificate game-ssl-cert -n game-2048

# Check challenges
kubectl get challenges -n game-2048

# View cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager --tail=200
```

### Pod Issues

```bash
# Check pod status
kubectl get pods -n game-2048

# Describe pod
kubectl describe pod <pod-name> -n game-2048

# View logs
kubectl logs <pod-name> -n game-2048
```

### Ingress Issues

```bash
# Check ingress
kubectl describe ingress ingress-2048 -n game-2048

# View NGINX logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

## 🔄 Updates

### Update Application

```bash
# Update image
kubectl set image deployment/deployment-2048 \
  app-2048=NEW_IMAGE:TAG \
  -n game-2048

# Watch rollout
kubectl rollout status deployment/deployment-2048 -n game-2048
```

### Rollback

```bash
kubectl rollout undo deployment/deployment-2048 -n game-2048
```

## 🧹 Cleanup

```bash
# Delete application
kubectl delete -f k8s/

# Delete NGINX Ingress
helm uninstall ingress-nginx -n ingress-nginx

# Delete cert-manager
helm uninstall cert-manager -n cert-manager
```

## 🏢 Enterprise Features

1. **High Availability**: Multi-zone deployment with PDB
2. **Auto-scaling**: HPA for pods and NGINX controller
3. **Security**: Network policies, Pod Security Standards, rate limiting
4. **Reliability**: Health checks, rolling updates, graceful shutdown
5. **Observability**: Labels, annotations, monitoring ready
6. **Resource Management**: Quotas, limits, and requests
7. **Storage**: Regional persistent disks with retention policy

## 🔐 Security Considerations

- All pods run as non-root user (UID 65534)
- Capabilities dropped, only NET_BIND_SERVICE added
- Security context with seccomp profile
- Network policies restrict traffic flow
- TLS certificates automatically renewed
- Rate limiting and connection limits enabled
- Security headers set on all responses

## 📝 Migration from GCE Ingress

### Key Changes

1. **Ingress Controller**: GCE → NGINX
2. **Certificate Management**: ManagedCertificate → cert-manager
3. **Service Type**: NodePort → ClusterIP
4. **Static IP**: Not required with NGINX
5. **TLS Configuration**: Automated with cert-manager

### Migration Benefits

- Better performance and flexibility
- Advanced routing capabilities
- Cross-cloud portability
- Open-source ecosystem
- More configuration options

## 📚 References

- [NGINX Ingress Documentation](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager Documentation](https://cert-manager.io/)
- [GKE Autopilot Guide](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)

## 📄 License

This deployment configuration is provided as-is for educational and production use.

## 🤝 Contributing

Contributions welcome! Please ensure all changes maintain enterprise-grade standards and security best practices.

## 📧 Support

For issues or questions:
1. Check the troubleshooting section in DEPLOYMENT-GUIDE.md
2. Review NGINX Ingress and cert-manager documentation
3. Check Kubernetes events: `kubectl get events -n game-2048`

---

**Built for enterprise production on GKE Autopilot** 🚀

# Grafana Ingress Configuration

This component configures an Ingress resource for Grafana with SSL/TLS support using Google Cloud Managed Certificates and a static IP address.

## Overview

This component provides:
- **Ingress**: GCE Ingress Controller configuration for external access
- **Static IP**: Reserved global static IP address for stable DNS configuration
- **SSL/TLS**: Google Cloud Managed Certificate for automatic HTTPS
- **DNS Ready**: Configuration ready for DNS setup

## Prerequisites

- A GKE (Google Kubernetes Engine) cluster
- Grafana installed in the `monitoring` namespace
- `kubectl` configured to access your cluster
- `gcloud` CLI installed and configured
- A domain name for Grafana (e.g., `grafana.drdroid.akhilphilip.com`)
- DNS management access (Cloudflare, Google Cloud DNS, or your DNS provider)

## Installation Steps

### Step 1: Create Static IP Address

Create a global static IP address for the Grafana Ingress:

```bash
gcloud compute addresses create grafana-monitor-ip --global
```

Verify the IP was created:

```bash
gcloud compute addresses describe grafana-monitor-ip --global --format="get(address)"
```

**Note**: Save this IP address - you'll need it for DNS configuration.

### Step 2: Create Managed Certificate

Create a Google Cloud Managed Certificate for SSL/TLS:

```bash
cat > managed-certificate.yaml <<'YAML'
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: grafana-cert
  namespace: monitoring
spec:
  domains:
    - grafana.drdroid.akhilphilip.com
YAML

kubectl apply -f managed-certificate.yaml
```

**Note**: Replace `grafana.drdroid.akhilphilip.com` with your actual domain name.

Monitor certificate provisioning status:

```bash
kubectl describe managedcertificate grafana-cert -n monitoring
```

Certificate provisioning typically takes 10-60 minutes. Wait until the status shows `Active`.

### Step 3: Update Ingress Configuration

Before applying the ingress, update the hostname in `ingress.yaml`:

```bash
# Edit the ingress.yaml file and update the host field
# Change: grafana.drdroid.akhilphilip.com to your domain
```

Apply the ingress:

```bash
kubectl apply -f ingress.yaml
```

### Step 4: Wait for Ingress IP Assignment

The Ingress will take 5-10 minutes to provision and assign the static IP. Monitor the status:

```bash
kubectl get ingress grafana -n monitoring
```

Or watch for the IP to appear:

```bash
watch -n 10 'kubectl get ingress grafana -n monitoring'
```

Once the `ADDRESS` column shows an IP, proceed to DNS configuration.

**Alternative**: Check the Ingress details:

```bash
kubectl describe ingress grafana -n monitoring
```

### Step 5: Configure DNS

Point your domain to the Ingress IP address. The IP address can be found using:

```bash
# Get the static IP (if Ingress hasn't assigned it yet)
gcloud compute addresses describe grafana-monitor-ip --global --format="get(address)"

# Or get the Ingress IP (once assigned)
kubectl get ingress grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

#### Option A: Google Cloud DNS

```bash
gcloud dns record-sets create grafana.drdroid.akhilphilip.com. \
  --rrdatas=<INGRESS_IP> \
  --type=A \
  --ttl=300 \
  --zone=<your-dns-zone-name>
```

#### Option B: Cloudflare DNS

1. Log in to Cloudflare Dashboard
2. Select your domain (`drdroid.akhilphilip.com`)
3. Go to **DNS** → **Records**
4. Add or edit the A record:
   - **Type**: A
   - **Name**: `grafana` (or `grafana.drdroid` depending on your setup)
   - **IPv4 address**: `<INGRESS_IP>` (from Step 4)
   - **Proxy status**: 
     - **DNS only** (gray cloud) - Recommended for initial testing
     - **Proxied** (orange cloud) - Use if you want Cloudflare features (CDN, DDoS protection, etc.)

**Important**: If using Cloudflare proxy, ensure:
- SSL/TLS mode is set to **Full** or **Full (strict)** in Cloudflare SSL/TLS settings
- The origin server (Ingress IP) is reachable from Cloudflare's network

#### Option C: Other DNS Providers

Create an A record:
- **Name**: `grafana` (or your subdomain)
- **Type**: A
- **Value**: `<INGRESS_IP>`
- **TTL**: 300 seconds (or your preference)

### Step 6: Verify Access

Wait 1-2 minutes for DNS propagation, then test:

```bash
# Test HTTP access
curl -I http://grafana.drdroid.akhilphilip.com

# Test HTTPS access (once certificate is active)
curl -I https://grafana.drdroid.akhilphilip.com
```

You should see HTTP 302 (redirect) or HTTP 200 responses.

## Troubleshooting

### Ingress IP Not Appearing

If the Ingress doesn't get an IP address after 10-15 minutes:

1. **Check Ingress status**:
   ```bash
   kubectl describe ingress grafana -n monitoring
   ```

2. **Verify static IP exists**:
   ```bash
   gcloud compute addresses list --global --filter="name=grafana-monitor-ip"
   ```

3. **Check for errors**:
   ```bash
   kubectl get events -n monitoring --sort-by='.lastTimestamp' | grep grafana
   ```

4. **Verify Grafana service exists**:
   ```bash
   kubectl get svc grafana -n monitoring
   ```

### Cloudflare 520 Error

If you see a **520 error** when accessing through Cloudflare:

**Symptoms**:
```
HTTP/1.1 520 <none>
Server: cloudflare
```

**Causes**:
- Cloudflare proxy is enabled but can't reach the origin server
- DNS points to Cloudflare IPs instead of your origin IP
- SSL/TLS mode mismatch

**Solutions**:

1. **Temporarily disable Cloudflare proxy** (quick test):
   - In Cloudflare DNS, set the A record to **DNS only** (gray cloud)
   - Wait 1-2 minutes and test again

2. **Verify origin IP is correct**:
   ```bash
   # Get the actual Ingress IP
   kubectl get ingress grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
   
   # Verify it matches your DNS A record
   dig +short grafana.drdroid.akhilphilip.com
   ```

3. **Check Cloudflare SSL/TLS settings**:
   - Go to Cloudflare Dashboard → SSL/TLS
   - Set encryption mode to **Full** or **Full (strict)**
   - Ensure the origin server accepts HTTPS connections

4. **Test direct IP access**:
   ```bash
   # Test if Grafana is reachable directly via IP
   curl -I http://<INGRESS_IP> -H "Host: grafana.drdroid.akhilphilip.com"
   ```

### Certificate Not Provisioning

If the managed certificate doesn't become active:

1. **Check certificate status**:
   ```bash
   kubectl describe managedcertificate grafana-cert -n monitoring
   ```

2. **Verify DNS is pointing to the Ingress IP**:
   ```bash
   dig +short grafana.drdroid.akhilphilip.com
   # Should return the Ingress IP
   ```

3. **Check for domain verification issues**:
   - Google Cloud needs to verify domain ownership
   - Ensure DNS A record is correctly configured
   - Wait 10-60 minutes for verification

4. **Common issues**:
   - DNS not propagated yet
   - Wrong IP address in DNS
   - Domain not accessible from the internet

### Service Not Responding

If the service returns errors:

1. **Check Grafana pods**:
   ```bash
   kubectl get pods -n monitoring -l app=grafana
   ```

2. **Check service endpoints**:
   ```bash
   kubectl get endpoints grafana -n monitoring
   ```

3. **Test service directly**:
   ```bash
   kubectl port-forward -n monitoring svc/grafana 3000:3000
   # Then access http://localhost:3000
   ```

4. **Check service configuration**:
   ```bash
   kubectl get svc grafana -n monitoring -o yaml
   ```

### Deprecated Annotation Warning

If you see a warning about deprecated `kubernetes.io/ingress.class`:

```
Warning: annotation "kubernetes.io/ingress.class" is deprecated, please use 'spec.ingressClassName' instead
```

This has been fixed in the current `ingress.yaml` which uses `spec.ingressClassName: gce` instead of the deprecated annotation.

## Configuration Details

### Ingress Configuration

The ingress uses:
- **Ingress Class**: `gce` (Google Cloud Load Balancer)
- **Static IP**: `grafana-monitor-ip` (global static IP)
- **Managed Certificate**: `grafana-cert` (automatic SSL/TLS)
- **Backend Service**: `grafana` service on port `3000`
- **Path**: `/` (all paths)

### Files

- `ingress.yaml`: Kubernetes Ingress resource configuration
- `managed-certificate.yaml`: Google Cloud Managed Certificate (create this file as shown in Step 2)

## Verification Checklist

- [ ] Static IP created and verified
- [ ] Managed certificate created and active
- [ ] Ingress applied and has an IP address
- [ ] DNS A record points to Ingress IP
- [ ] HTTP access works (curl test)
- [ ] HTTPS access works (after certificate provisioning)
- [ ] Grafana UI accessible via domain name

## Additional Resources

- [GKE Ingress Documentation](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress)
- [Google Cloud Managed Certificates](https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs)
- [Cloudflare DNS Setup](https://developers.cloudflare.com/dns/manage-dns-records/)


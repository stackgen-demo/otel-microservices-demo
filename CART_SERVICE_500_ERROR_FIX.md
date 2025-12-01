# Cart Service 500 Error Fix

## Problem
Getting `500 Internal Server Error` when accessing:
```
http://demo-store.stackgen.com/api/cart?sessionId=8c471bb8-f46b-459a-88e1-f856ebb0c58d&currencyCode=USD
```

## Root Cause Analysis

The request flow is:
1. **Client** → `http://demo-store.stackgen.com/api/cart` 
2. **Frontend-Proxy (Envoy)** → Routes `/api/*` to `frontend` service (catch-all route)
3. **Frontend Service** → Calls `cart:8080` service internally
4. **Cart Service** → Should respond, but returning 500 error

## Solution Implemented

### 1. Created ConfigMap for Cart Service Configuration

**File**: `demo-apps/frontend-proxy-service/templates/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-proxy-cart-config
data:
  CART_HOST: "cart"
  CART_PORT: "8080"
  CART_SERVICE_URL: "http://cart:8080"
```

### 2. Updated Deployment to Use ConfigMap

**File**: `demo-apps/frontend-proxy-service/templates/deployment.yaml`

Added environment variables from ConfigMap:
```yaml
env:
  - name: CART_HOST
    valueFrom:
      configMapKeyRef:
        name: frontend-proxy-cart-config
        key: CART_HOST
  - name: CART_PORT
    valueFrom:
      configMapKeyRef:
        name: frontend-proxy-cart-config
        key: CART_PORT
  - name: CART_SERVICE_URL
    valueFrom:
      configMapKeyRef:
        name: frontend-proxy-cart-config
        key: CART_SERVICE_URL
```

### 3. Updated values.yaml

Added cart service configuration to `values.yaml`:
```yaml
env:
  CART_HOST: "cart"
  CART_PORT: "8080"
  CART_SERVICE_URL: "http://cart:8080"
```

## Troubleshooting Steps

### 1. Check Cart Service Status

```bash
# Access cluster (use AWS Console link provided)
kubectl get pods -n opsverse-demo -l app.kubernetes.io/component=cart
kubectl get svc -n opsverse-demo cart
```

### 2. Check Frontend Service Logs

The frontend service is the one actually calling the cart service:

```bash
# Get frontend service logs
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=frontend --tail=100 | grep -i cart

# Check for connection errors
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=frontend --tail=100 | grep -i "error\|500\|failed"
```

### 3. Check Cart Service Logs

```bash
# Get cart service logs
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=cart --tail=100

# Check for errors
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=cart --tail=100 | grep -i "error\|exception\|failed"
```

### 4. Verify Cart Service Connectivity

```bash
# Test cart service from frontend pod
kubectl exec -n opsverse-demo -it $(kubectl get pod -n opsverse-demo -l app.kubernetes.io/component=frontend -o jsonpath='{.items[0].metadata.name}') -- \
  curl -v http://cart:8080/health || echo "Cart service not reachable"

# Check if valkey-cart (Redis) is accessible
kubectl get pods -n opsverse-demo -l app.kubernetes.io/component=valkey-cart
kubectl get svc -n opsverse-demo valkey-cart
```

### 5. Check Frontend-Proxy Logs

```bash
# Get frontend-proxy logs
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=frontend-proxy --tail=100

# Check Envoy access logs
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=frontend-proxy --tail=100 | grep "/api/cart"
```

## Common Issues and Fixes

### Issue 1: Cart Service Not Running
**Symptom**: Pod not found or in CrashLoopBackOff
**Fix**: 
```bash
kubectl describe pod -n opsverse-demo -l app.kubernetes.io/component=cart
kubectl logs -n opsverse-demo -l app.kubernetes.io/component=cart
```

### Issue 2: Valkey-Cart (Redis) Not Available
**Symptom**: Cart service can't connect to Redis
**Fix**: Ensure `valkey-cart-service` is deployed and running:
```bash
kubectl get pods -n opsverse-demo -l app.kubernetes.io/component=valkey-cart
kubectl get svc -n opsverse-demo valkey-cart
```

### Issue 3: DNS Resolution Failure
**Symptom**: `cart:8080` not resolving
**Fix**: Verify Service exists:
```bash
kubectl get svc -n opsverse-demo cart
# Should show:
# NAME   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
# cart   ClusterIP   <IP>         <none>        8080/TCP   <age>
```

### Issue 4: Network Policy Blocking
**Symptom**: Connection timeout
**Fix**: Check NetworkPolicies:
```bash
kubectl get networkpolicies -n opsverse-demo
```

## Verification

After deploying the changes:

1. **Verify ConfigMap exists**:
   ```bash
   kubectl get configmap -n opsverse-demo frontend-proxy-cart-config
   kubectl describe configmap -n opsverse-demo frontend-proxy-cart-config
   ```

2. **Verify Environment Variables**:
   ```bash
   kubectl exec -n opsverse-demo -it $(kubectl get pod -n opsverse-demo -l app.kubernetes.io/component=frontend-proxy -o jsonpath='{.items[0].metadata.name}') -- \
     env | grep CART
   ```

3. **Test Cart Service Directly**:
   ```bash
   # Port-forward to cart service
   kubectl port-forward -n opsverse-demo svc/cart 8080:8080
   
   # In another terminal, test
   curl http://localhost:8080/health
   curl "http://localhost:8080/api/cart?sessionId=test&currencyCode=USD"
   ```

4. **Test Through Frontend-Proxy**:
   ```bash
   # Port-forward to frontend-proxy
   kubectl port-forward -n opsverse-demo svc/frontend-proxy 8080:8080
   
   # Test cart endpoint
   curl "http://localhost:8080/api/cart?sessionId=test&currencyCode=USD"
   ```

## Next Steps

1. **Deploy the updated Helm chart** via ArgoCD
2. **Monitor logs** for cart service errors
3. **Check cart service health** endpoint
4. **Verify valkey-cart** (Redis) connectivity
5. **Test the endpoint** after deployment

## Additional Notes

- The frontend-proxy uses Envoy and routes `/api/*` requests to the `frontend` service
- The `frontend` service then makes HTTP calls to `cart:8080` using `CART_ADDR: "cart:8080"`
- The ConfigMap ensures frontend-proxy has cart service information available
- The actual cart service call is made by the frontend service, not the frontend-proxy

## Accessing Cluster Logs

To access the cluster and check logs, use the AWS Console:
1. Go to: https://us-east-1.console.aws.amazon.com/eks/clusters/eks-demo-eastus?region=us-east-1&selectedTab=cluster-access-tab
2. Follow the instructions to configure `kubectl`
3. Then run the troubleshooting commands above


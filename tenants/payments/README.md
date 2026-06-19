# Tenant: Payments

Multi-tenant setup cho team **payments** - cô lập an toàn với team **demo**.

## Cấu hình

### 1. Namespace
- Name: `payments`
- Labels: 
  - `tenant: payments`
  - `policy.sigstore.dev/include: "true"` (opt-in Policy Controller)

### 2. RBAC (Least Privilege)
- **Role:** `payments-developer`
- **User:** `payments-dev`
- **Quyền:**
  - ✅ Tạo/sửa/xóa: Deployment, Pod, Service, ConfigMap, Rollout
  - ✅ Đọc: Secrets
  - ❌ Tạo/sửa/xóa: Secrets, Roles, RoleBindings

### 3. ResourceQuota
- CPU requests: 2 cores
- Memory requests: 4Gi
- CPU limits: 4 cores
- Memory limits: 8Gi
- Pods: 10
- Services: 5

### 4. LimitRange
- **Default limits** (cho pod không khai báo):
  - CPU: 500m
  - Memory: 512Mi
- **Max per container:**
  - CPU: 2 cores
  - Memory: 2Gi

### 5. NetworkPolicy
- **Default deny ingress:** Chặn tất cả traffic vào
- **Egress whitelist:**
  - ✅ DNS (kube-system:53)
  - ✅ Service `api` trong namespace `demo` (port 8080)
  - ❌ Tất cả egress khác bị chặn

## Kiểm tra RBAC

```powershell
# payments-dev có thể tạo deployment trong payments
kubectl auth can-i create deploy -n payments --as payments-dev

# payments-dev KHÔNG thể tạo secrets trong payments
kubectl auth can-i create secrets -n payments --as payments-dev

# payments-dev KHÔNG thể tạo rolebindings trong payments
kubectl auth can-i create rolebindings -n payments --as payments-dev

# payments-dev KHÔNG thể làm gì trong namespace demo
kubectl auth can-i create deploy -n demo --as payments-dev
```

## Kiểm tra ResourceQuota

```powershell
# Xem quota hiện tại
kubectl get resourcequota -n payments

# Describe để xem usage
kubectl describe resourcequota payments-quota -n payments
```

### Test: Pod vượt quota bị reject

```powershell
# Pod xin quá nhiều RAM (vượt quota 8Gi)
@"
apiVersion: v1
kind: Pod
metadata:
  name: test-exceed-quota
  namespace: payments
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: test
    image: nginx:1.25
    resources:
      limits:
        cpu: 100m
        memory: 10Gi
"@ | kubectl apply --dry-run=server -f -
# Expected: Error (exceeds quota)
```

## Kiểm tra LimitRange

```powershell
# Xem LimitRange
kubectl get limitrange -n payments
kubectl describe limitrange payments-limitrange -n payments
```

### Test: Pod không khai limits được default

```powershell
# Pod không khai resources → LimitRange cấp default
@"
apiVersion: v1
kind: Pod
metadata:
  name: test-no-limits
  namespace: payments
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: test
    image: nginx:1.25
"@ | kubectl apply --dry-run=server -f -
# Expected: Created (LimitRange tự động thêm limits)

# Kiểm tra pod đã có limits
kubectl get pod test-no-limits -n payments -o yaml | grep -A 4 resources
```

## Kiểm tra NetworkPolicy

```powershell
# Xem NetworkPolicy
kubectl get networkpolicy -n payments

# Test từ payments pod gọi sang demo
kubectl run test-netpol --image=curlimages/curl:latest --rm -i -n payments -- \
  curl -s --max-time 5 http://api.demo.svc:8080/health

# Expected: Cho phép (nếu CNI enforce)
```

## Kiểm tra Gatekeeper Policy (constraint cũ)

App hợp lệ (đã ký + đủ limits) chạy xanh:
```powershell
kubectl get pods -n payments
```

Manifest vi phạm bị chặn:
```powershell
# Test: Pod dùng :latest tag
@"
apiVersion: v1
kind: Pod
metadata:
  name: test-latest
  namespace: payments
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: test
    image: nginx:latest
    resources:
      limits:
        cpu: 100m
        memory: 128Mi
"@ | kubectl apply --dry-run=server -f -
# Expected: Rejected by disallow-latest-tag constraint
```

# TODO: put everything in a kustomization.yaml file

To deploy Crossplane’s **`provider-kubernetes`** in one shot, ensure your YAML file includes all required components in the correct order. Here’s a breakdown of the **required resources** and their dependencies:

---

### **1. Required Components**
1. **`Provider`**: Installs the Kubernetes provider.
2. **`ServiceAccount`**: For authentication to the target cluster(s).
3. **`ClusterRole`**: Grants permissions to manage Kubernetes resources.
4. **`ClusterRoleBinding`**: Binds the `ServiceAccount` to the `ClusterRole`.
5. **`ProviderConfig`**: Configures how the provider interacts with the target cluster(s).

---

### **Full YAML Example (Order Matters!)**
```yaml
# 1. Install the Kubernetes Provider
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: "xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.10.0"  # Check for latest version
---
# 2. Create a ServiceAccount for the Provider
apiVersion: v1
kind: ServiceAccount
metadata:
  name: crossplane-kubernetes-provider
  namespace: crossplane-system  # Must match Crossplane's install namespace
---
# 3. Define ClusterRole Permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: crossplane-kubernetes-provider
rules:
- apiGroups: ["*"]
  resources: ["*"]  # Adjust to least privilege for your use case
  verbs: ["*"]
---
# 4. Bind ClusterRole to ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: crossplane-kubernetes-provider
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: crossplane-kubernetes-provider
subjects:
- kind: ServiceAccount
  name: crossplane-kubernetes-provider
  namespace: crossplane-system
---
# 5. Configure ProviderConfig (to manage the host cluster)
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity  # Uses the ServiceAccount's token
```

---

### **Key Notes**
#### **1. Order of Resources**  
The YAML **must** be applied in this order to avoid dependency issues:
1. `Provider` (installs the provider).
2. `ServiceAccount` (identity for the provider).
3. `ClusterRole` + `ClusterRoleBinding` (grants permissions).
4. `ProviderConfig` (configures the provider).

#### **2. Missing Parameters**  
Common missing pieces include:
- **`spec.credentials.source`** in `ProviderConfig` (must be `InjectedIdentity`, `Secret`, or `Filesystem`).
- **RBAC Rules**: Overly restrictive `ClusterRole` rules (start with `["*"]` for testing).
- **Namespace**: The `ServiceAccount` must be in the same namespace as Crossplane (`crossplane-system` by default).

#### **3. Alternative: Use `kubeconfig`**  
To manage **external clusters**, replace the `ProviderConfig` with a `Secret`-based configuration:
```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: external-cluster
spec:
  credentials:
    source: Secret
    secretRef:
      name: external-cluster-kubeconfig  # Secret containing `kubeconfig`
      namespace: crossplane-system
      key: kubeconfig
```

---

### **Verification**
After applying the YAML:
```bash
# Check if the provider is HEALTHY
kubectl get providers

# Verify ProviderConfig status
kubectl get providerconfig
```

---

### **Troubleshooting**  
- **Permissions Issues**: Check the provider logs:  
  ```bash
  kubectl logs -n crossplane-system \
    $(kubectl get pods -n crossplane-system -o name | grep provider-kubernetes)
  ```
- **ProviderConfig Not Ready**: Ensure the `ServiceAccount` and `ClusterRoleBinding` exist.
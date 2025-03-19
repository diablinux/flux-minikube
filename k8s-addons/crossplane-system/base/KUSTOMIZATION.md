To deploy the Crossplane `provider-kubernetes` using Kustomize, you’ll organize your YAML manifests into a Kustomize structure and ensure dependencies are resolved. Here’s how to do it:

---

### **Step 1: Create a Kustomize Directory Structure**
```plaintext
.
├── 📁 base
│   ├── 📁 provider
│   │   ├── 📄 provider.yaml          # Provider resource
│   │   ├── 📄 service-account.yaml   # ServiceAccount
│   │   ├── 📄 cluster-role.yaml      # ClusterRole
│   │   ├── 📄 cluster-role-binding.yaml # ClusterRoleBinding
│   │   └── 📄 provider-config.yaml   # ProviderConfig
│   └── 📄 kustomization.yaml         # Base kustomization
└── 📁 overlays
    └── 📁 production                 # Example overlay for a specific environment
        ├── 📄 kustomization.yaml
        └── 📄 patch.yaml             # Optional patches (e.g., external cluster config)
```

---

### **Step 2: Define Base Files**
#### 1. `base/provider/provider.yaml`
```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.10.0
```

#### 2. `base/provider/service-account.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: crossplane-kubernetes-provider
  namespace: crossplane-system
```

#### 3. `base/provider/cluster-role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: crossplane-kubernetes-provider
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

#### 4. `base/provider/cluster-role-binding.yaml`
```yaml
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
```

#### 5. `base/provider/provider-config.yaml`
```yaml
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: InjectedIdentity  # For managing the local cluster
```

---

### **Step 3: Define the Base Kustomization**
#### `base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - provider/provider.yaml
  - provider/service-account.yaml
  - provider/cluster-role.yaml
  - provider/cluster-role-binding.yaml
  - provider/provider-config.yaml
```

---

### **Step 4: Optional Overlay for Customization**
#### Example: Overlay for an external cluster (`overlays/production/kustomization.yaml`)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../base
patches:
  - target:
      kind: ProviderConfig
      name: default
    patch: |-
      - op: replace
        path: /spec/credentials/source
        value: Secret
      - op: replace
        path: /spec/credentials/secretRef
        value:
          name: external-cluster-kubeconfig
          namespace: crossplane-system
          key: kubeconfig
# If you need to add a Secret for kubeconfig:
secretGenerator:
  - name: external-cluster-kubeconfig
    namespace: crossplane-system
    files:
      - kubeconfig=./kubeconfig.yaml  # Path to your kubeconfig file
    type: Opaque
```

---

### **Step 5: Apply the Configuration**
```bash
# Apply the base configuration (local cluster)
kubectl apply -k base/

# Apply the overlay for an external cluster
kubectl apply -k overlays/production/
```

---

### **Key Benefits of Kustomize**
1. **Order Management**: Kustomize applies resources in the order they’re listed in `resources`, ensuring dependencies are resolved.  
2. **Reusability**: Base configurations can be reused across environments (dev, staging, prod).  
3. **Patch Flexibility**: Modify the `ProviderConfig` for different clusters without duplicating YAML.  

---

### **Verification**
```bash
# Check if the provider is healthy
kubectl get providers

# Verify ProviderConfig
kubectl get providerconfig
```

---

### **Troubleshooting**
- **Missing Permissions**: Ensure the `ClusterRole` grants sufficient access.  
- **Secret Issues**: If using an external cluster, validate the `kubeconfig` in the Secret.  
- **Provider Logs**:  
  ```bash
  kubectl logs -n crossplane-system \
    $(kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-kubernetes -o name)
  ```

---

This Kustomize setup ensures a clean, repeatable deployment of `provider-kubernetes` with minimal manual intervention. 
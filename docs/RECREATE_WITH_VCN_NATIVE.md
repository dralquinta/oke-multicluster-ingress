# Recreating OKE Clusters with VCN-Native Pod Networking

## Why VCN-Native Networking?

The current clusters use **Flannel overlay networking**, which creates an isolated pod network (10.244.0.0/16 and 10.245.0.0/16) that doesn't integrate with OCI VCN routing. This prevents cross-cluster pod-to-pod connectivity through DRG/RPC.

**VCN-native pod networking** assigns pod IPs directly from VCN subnets, making them routable through standard VCN routing (VCN route tables, DRG, RPC).

## Current Status

- ✅ Both clusters deleted (in progress)
- ✅ VCN infrastructure preserved (VCNs, DRG, RPC, security lists, route tables)
- ✅ Network configuration ready for VCN-native pods

## Steps to Recreate Clusters

### Option 1: OCI Console (Recommended - Easier)

#### Primary Cluster (us-sanjose-1)

1. Navigate to **Developer Services** → **Kubernetes Clusters (OKE)**
2. Click **Create Cluster** → **Quick Create**
3. Configuration:
   - **Name**: `primary-cluster`
   - **Compartment**: current compartment
   - **Kubernetes version**: v1.34.1
   - **Kubernetes API endpoint**: Public endpoint
   - **Node type**: Managed
   - **Kubernetes worker nodes**: Public workers
   - **Shape**: VM.Standard.E4.Flex (or current shape)
   - **Number of nodes**: 3
   - **Network type**: **OCI VCN-Native Pod Networking** ⚠️ CRITICAL
   - **VCN**: Select existing `oke-vcn-quick-dalquint-oke-cluster-*` (10.0.0.0/16)
   - **Pod CIDR**: 10.244.0.0/16 (will create pod subnet automatically)

4. Click **Next** → **Create Cluster**
5. Wait ~10 minutes for cluster creation

#### Secondary Cluster (us-chicago-1)

1. Switch region to **us-chicago-1**
2. Navigate to **Developer Services** → **Kubernetes Clusters (OKE)**
3. Click **Create Cluster** → **Quick Create**
4. Configuration:
   - **Name**: `secondary-cluster`
   - **Compartment**: current compartment
   - **Kubernetes version**: v1.34.1
   - **Kubernetes API endpoint**: Public endpoint
   - **Node type**: Managed
   - **Kubernetes worker nodes**: Public workers
   - **Shape**: VM.Standard.E4.Flex
   - **Number of nodes**: 3
   - **Network type**: **OCI VCN-Native Pod Networking** ⚠️ CRITICAL
   - **VCN**: Select existing `dalquint-vcn` (10.1.0.0/16)
   - **Pod CIDR**: 10.245.0.0/16 (will create pod subnet automatically)

5. Click **Next** → **Create Cluster**
6. Wait ~10 minutes for cluster creation

### Option 2: OCI CLI (Advanced)

```bash
# Wait for deletions to complete first
# Then create primary cluster with VCN-native networking

# Create primary cluster
oci ce cluster create \
  --compartment-id ocid1.compartment.oc1..aaaaaaaal7vn7wsy3qgizklrlfgo2vllfta3wkqlnfkvykoroite3lzxbnna \
  --name "primary-cluster" \
  --kubernetes-version "v1.34.1" \
  --vcn-id ocid1.vcn.oc1.us-sanjose-1.amaaaaaafioir7ia5xm6yb5kpw7cz3cv6wtf6znot3xqxfqnp4q7qd3emoda \
  --cluster-pod-network-options '[{"cni-type":"OCI_VCN_IP_NATIVE"}]' \
  --region us-sanjose-1

# Create secondary cluster  
oci ce cluster create \
  --compartment-id ocid1.compartment.oc1..aaaaaaaal7vn7wsy3qgizklrlfgo2vllfta3wkqlnfkvykoroite3lzxbnna \
  --name "secondary-cluster" \
  --kubernetes-version "v1.34.1" \
  --vcn-id ocid1.vcn.oc1.us-chicago-1.amaaaaaafioir7iay743pltrniy5qndhqze2qxa57rzhshopbfm7zsorb46a \
  --cluster-pod-network-options '[{"cni-type":"OCI_VCN_IP_NATIVE"}]' \
  --region us-chicago-1
```

## After Cluster Creation

### 1. Update Kubeconfig

```bash
# Primary cluster
oci ce cluster create-kubeconfig \
  --cluster-id <NEW_PRIMARY_CLUSTER_ID> \
  --file ~/.kube/config \
  --region us-sanjose-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

# Secondary cluster  
oci ce cluster create-kubeconfig \
  --cluster-id <NEW_SECONDARY_CLUSTER_ID> \
  --file ~/.kube/config \
  --region us-chicago-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

# Rename contexts
kubectl config rename-context <context1> primary-cluster-context
kubectl config rename-context <context2> secondary-cluster
```

### 2. Verify VCN-Native Networking

```bash
# Check primary cluster
oci ce cluster get --cluster-id <PRIMARY_ID> --region us-sanjose-1 | \
  jq '.data."cluster-pod-network-options"'
# Should show: [{"cni-type": "OCI_VCN_IP_NATIVE"}]

# Check secondary cluster
oci ce cluster get --cluster-id <SECONDARY_ID> --region us-chicago-1 | \
  jq '.data."cluster-pod-network-options"'
# Should show: [{"cni-type": "OCI_VCN_IP_NATIVE"}]
```

### 3. Update Security Lists for Pod Subnets

The quick-create will create new pod subnets. You'll need to update their security lists to allow cross-cluster traffic:

```bash
# List the new pod subnets
oci network subnet list \
  --compartment-id ocid1.compartment.oc1..aaaaaaaal7vn7wsy3qgizklrlfgo2vllfta3wkqlnfkvykoroite3lzxbnna \
  --vcn-id <VCN_ID> \
  --region <REGION> | \
  jq '.data[] | select(.["display-name"] | contains("pod")) | {name, cidr, security_lists}'

# Add 0.0.0.0/0 ingress rule to pod subnet security lists (as we did before)
```

### 4. Test Pod-to-Pod Connectivity

```bash
# Deploy test pods
kubectl --context=primary-cluster-context run test-pod --image=busybox --command -- sleep 3600
kubectl --context=secondary-cluster run test-pod --image=busybox --command -- sleep 3600

# Get pod IPs (should now be in VCN CIDR ranges, not 10.244/10.245)
kubectl --context=primary-cluster-context get pod test-pod -o wide
kubectl --context=secondary-cluster get pod test-pod -o wide

# Test connectivity
kubectl --context=primary-cluster-context exec test-pod -- ping -c 3 <SECONDARY_POD_IP>
```

## Expected Behavior with VCN-Native Networking

- ✅ Pod IPs will be allocated from VCN subnets (not overlay network)
- ✅ Pods will be directly routable through VCN route tables
- ✅ DRG routes will apply to pod traffic automatically
- ✅ Cross-cluster pod-to-pod connectivity will work without additional configuration
- ✅ No need for DaemonSets to add routes on nodes

## Network Infrastructure (Already Configured)

All network infrastructure is already in place:
- ✅ DRG in both regions
- ✅ RPC peered between regions
- ✅ VCN route tables with DRG routes (10.244.0.0/16 ↔ 10.245.0.0/16)
- ✅ DRG route tables with pod CIDR static routes
- ✅ Security lists allowing 0.0.0.0/0 traffic
- ✅ Import/export route distributions configured

Once clusters are recreated with VCN-native networking, pod traffic will flow:
**Pod A (Primary)** → **VCN Route Table** → **DRG** → **RPC** → **DRG** → **VCN Route Table** → **Pod B (Secondary)**

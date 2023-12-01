# vCluster

## Summary

Provide a sandbox for developers and or an environment to test service configuration.

Initial scope: Deploy Gloo Mesh v2.4.5 using vCluster on EKS.

## Setup

### Perquisites
- [eksctl](https://eksctl.io/) 
- [vlcuster](https://www.vcluster.com/docs/getting-started/setup)
- [meshctl](https://docs.solo.io/gloo-mesh-enterprise/latest/setup/prepare/meshctl_cli_install/)

### vCluster
vCluster requires a persistent volume to store etc data. Please note the use of the EBS CSI driver in the eks/eks-config-east.yaml

```
eksctl create addon --name aws-ebs-csi-driver --cluster vcluster --service-account-role-arn arn:aws:iam::986112284769:role/AmazonEKS_EBS_CSI_DriverRole
```

```
vcluster list
vcluster connect my-vcluster -n vcluster-my-vcluster

kubectx
export CLUSTER_NAME=mgmt

# Alias to download and update meshctl version
mesh 2.4.5

# Alias to generate 30 day license
mkl

meshctl install --profiles gloo-mesh-single,ratelimit,extauth \
  --set common.cluster=$CLUSTER_NAME \
  --set demo.manageAddonNamespace=true \
  --set glooMgmtServer.createGlobalWorkspace=true \
  --set licensing.glooMeshLicenseKey=$GLOO_MESH_LICENSE_KEY
meshctl check

export REVISION=$(kubectl get pod -L app=istiod -n istio-system -o jsonpath='{.items[0].metadata.labels.istio\.io/rev}')\necho $REVISION

kubectl create ns bookinfo\nkubectl label ns bookinfo istio.io/rev=$REVISION --overwrite=true\n
# deploy bookinfo application components for all versions less than v3
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/istio/istio/1.18.3/samples/bookinfo/platform/kube/bookinfo.yaml -l 'app,version notin (v3)'

# deploy an updated product page with extra container utilities such as 'curl' and 'netcat'
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/solo-io/gloo-mesh-use-cases/main/policy-demo/productpage-with-curl.yaml

# deploy all bookinfo service accounts
kubectl -n bookinfo apply -f https://raw.githubusercontent.com/istio/istio/1.18.3/samples/bookinfo/platform/kube/bookinfo.yaml -l 'account'
kubectl get pods -n bookinfo\nkubectl get svc -n bookinfo
kubectl get pods -n bookinfo -w\nkubectl get svc -n bookinfo -w

export REVISION=$(kubectl get pod -L app=istiod -n istio-system -o jsonpath='{.items[0].metadata.labels.istio\.io/rev}')
echo $REVISION

kubectl create ns httpbin\nkubectl label ns httpbin istio.io/rev=$REVISION --overwrite=true
kubectl -n httpbin apply -f https://raw.githubusercontent.com/solo-io/gloo-mesh-use-cases/main/policy-demo/httpbin.yaml
kubectl -n httpbin get pods
kubectl -n httpbin get pods -w

kubectl create ns helloworld
kubectl label ns helloworld istio.io/rev=$REVISION --overwrite=true
kubectl -n helloworld apply -f https://raw.githubusercontent.com/solo-io/gloo-mesh-use-cases/main/policy-demo/helloworld.yaml
kubectl -n helloworld get pods -w

# port forward the bookinfo app
kubectl -n bookinfo port-forward deployment/productpage-v1 9080:9080
```

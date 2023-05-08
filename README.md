# statefulset demo
This example is referenced in the kubernetes documentation:
```
https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/
```

## provision a EKS cluster with CSI driver
This statefulset is require a cluser.
You could build your own cluster by following this pulumi_EKS repo
```
git clone https://github.com/one2cloudpeter/pulumi_EKS
cd pulumi_EKS
pulumi up
```

By provision a storage class for cluster storage, you would have to deploy a CSI driver also by following this tutorials, we are using us-east-1 region as an example
```
https://www.eksworkshop.com/beginner/170_statefulset/ebs_csi_driver/
```
```
# download the policy file
curl -sSL -o ebs-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json

# Create the IAM policy
aws iam create-policy --region us-east-1 --policy-name "Amazon_EKS_EBS_CSI_Driver" --policy-document <your downloaded policy location>

# Create an IAM OIDC provider for your cluster
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=<EKS_cluster_name> --approve

eksctl create iamserviceaccount --cluster <EKS_cluster_name> --name ebs-csi-controller-irsa --namespace kube-system --attach-policy-arn <Amazon_EKS_EBS_CSI_Driver_Policy_ARN> --override-existing-serviceaccounts --approve

# add the aws-ebs-csi-driver as a helm repo
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver


helm upgrade --install aws-ebs-csi-driver --version=1.2.4 --namespace kube-system --set serviceAccount.controller.create=false --set serviceAccount.snapshot.create=false --set enableVolumeScheduling=true --set enableVolumeResizing=true --set enableVolumeSnapshot=true  --set serviceAccount.snapshot.name=ebs-csi-controller-irsa --set serviceAccount.controller.name=ebs-csi-controller-irsa aws-ebs-csi-driver/aws-ebs-csi-driver

kubectl -n kube-system rollout status deployment ebs-csi-controller
```



## apply the storage class
```
kubectl apply -f storageclass.yml
```
once the storage class was ready
```
kubectl describe storageclass mysql-gp2
```
you could claim the pvc from the storage class as following yaml
```
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mysql-gp2
      resources:
        requests:
          storage: 10Gi
```

## deploy the example statefulset
```
kubectl apply -f statefulset.yml
```

```
kubectl describe statefulset
kubectl get pod -w
```

once the pod was ready, you could exec to one of the pod and create a html file for curl test
```
kubectl exec -it web-0 bash

cd /usr/share/nginx/html/
echo 'echo "Hello, this is a statefulset pod web-0" > /usr/share/nginx/html/index.html'
```

the expected output would be "Hello, this is a statefulset pod web-0" 
```
curl http://localhost/
```

exit the pod
```
exit
```

## delete the pod for testing
```
kubectl delete pod web-0
```

```
kubectl get pod -w
```
once the pod is rebuilded, you could curl the pod again, the expected output would be "Hello, this is a statefulset pod web-0", that mean the storage still exists after the pod is delete
```
kubectl exec -i -t "web-0" -- curl http://localhost/
```

## clean up

```
kubectl delete -f statefulset
```
since the pvc is reserved to prevent data loss, so after the statefulset is deleted, you would still have to delete the pvc manually
```
kubectl delete pvc --all
```
# K8s for Istio - Kiali - Jaeger

### Setting up infra on AWS for Istio, Kiali and Jaeger 

## Istio

Istio is an open-source service mesh platform that provides a way to control how microservices share data with one another. It includes APIs that let Istio integrate into any logging platform, telemetry, or policy system. Istio is designed to run in a variety of environments: on-premise, cloud-hosted, in a Kubernetes container, and in services running on Virtual Machines.

![image](https://github.com/Pavan-1997/K8s_Istio_Kiali_Jaeger/assets/32020205/7b068efa-a8ae-47b6-aa51-9499ec1b1cda)

## Kiali

Kiali works with Istio in Kubernetes distributions. It visualizes the service mesh topology and provides visibility into features like request routing, circuit breakers, request rates, latency and more.

In simple words, it gives the big picture of Routing/Networking.


## Jaeger

Jaeger is open-source software for tracing transactions between distributed services. It’s used for monitoring and troubleshooting complex microservices environments.

Problems that Jaegger addresses:-

- Distributed transaction monitoring
- Performance and latency optimization
- Root cause analysis
- Service dependency analysis
- Distributed context propagation

---
# Implementation

1. Create the AWS EC2 Linux AMI instance with `t2.medium` and login using `username:ec2-user`


2. Install kubectl (For accessing PODS and K8s resources)
```
sudo -i

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

echo "$(cat kubectl.sha256) kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version
```

3. Install eksctl (For creating the K8s cluster)
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/bin

eksctl version
```

4. Addiing IAM Role to EC2 Instance created in Step 1. for EC2 to access the AWS EKS 

    Go to IAM -> On the left pane click on Roles -> Click on Create Role -> Trusted entity type: AWS service -> Select Common use case: EC2 -> Click on Next -> Now select the Permission policies: AdministratorAccess -> Click on Next -> Give Role name -> Click on Create role
    
    Now go to EC2 instance -> Select the EC2 instance created -> Go to Actions on top right -> Select Security -> Modify IAM role -> Now select the IAM role created -> Click on Update IAM role


5. Create the K8s cluster
```
eksctl create cluster --name=eksdemo1 --region=us-west-1 --zones=us-west-1b,us-west-1a --without-nodegroup
```
![image](https://github.com/Pavan-1997/K8s_Istio_Kiali_Jaeger/assets/32020205/2bd11203-dd5b-4493-aa24-8eb21c9565c7)


6. Add OIDC
```
eksctl utils associate-iam-oidc-provider --region us-west-1 --cluster eksdemo1 --approve
```

7. Add Nodes
```
eksctl create nodegroup --cluster=eksdemo1 --region=us-west-1 --name=eksdemo-ng-public --node-type=t2.medium --nodes=2 --nodes-min=2 --nodes-max=4 --node-volume-size=10 --ssh-access --ssh-public-key=K8S_KEY --managed --asg-access --external-dns-access --full-ecr-access --appmesh-access --alb-ingress-access
```
![image](https://github.com/Pavan-1997/K8s_Istio_Kiali_Jaeger/assets/32020205/d649e5e5-b23d-4c9e-87da-8eee897d6b5f)


8. Check the Pods in Kube-System namespace
```
kubectl get pods -n kube-system
```

9. Install Istio
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.1 TARGET_ARCH=x86_64 sh -
```

10. Go into the directory (This contains Sample applications in samples/ and istioctl client binary in the bin/ directory)
```
cd istio-1.18.1
```

11. Set the Path
```
export PATH=$PWD/bin:$PATH
```

12. Install Istio with Demo Profile
```
istioctl install --set profile=demo -y
```

13. Check the Namespace in cluster
```
kubectl get namespace
```
![image](https://github.com/Pavan-1997/K8s_Istio_Kiali_Jaeger/assets/32020205/3bb1ddf2-3ce5-4f61-890f-c8f7dd17ee84)


14. Check Pods created in Istio Namespace
```
kubectl get pods -n istio-system
```

15. Deploy Application linked to Istio 
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/bookinfo/platform/kube/bookinfo.yaml
```

16. Checking the pods created from Step 15. in Default Namespace
```
kubectl get pods -n default
```
![image](https://github.com/Pavan-1997/K8s_Istio_Kiali_Jaeger/assets/32020205/8112e798-c7d6-4ada-bd5b-4e6e76aae429)


17. Checking the services
```
kubectl get service
```

18. Check our Product Page application is up and running 
```
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

19. Injection of Istio as an Init container (So that now 2 containers will run in every pod)
```
kubectl label namespace default istio-injection=enabled
```

20. Since every Pod in this cluster when deploying is created using Replica-set, To see the change in Conatiners from 1 to 2

`We have to basically delete the Pods individually and Replica-set will do its job again by re-creating the PODS with 2 Conatiners with has now extra of the Istio side car container`
```
kubectl delete pod <Pod1> <Pod1>
```

21. Now again checking the pods created in Default Namespace
```
kubectl get pods -n default
```

22. To analyze Istio is enabled on every Pod
```
istioctl analyze
```

23. Go to networking folder in samples and deploy the gateway manifest

`PWD -> /root/istio-1.18.1/samples/bookinfo/networking`
```
cd samples/bookinfo/networking/
```
```
kubectl apply -f bookinfo-gateway.yaml
```

24. Check the Gateway created from Step 23. 
```
kubectl get gateway
```

25. Set the ingress IP and ports
```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
```

26. Check the Port set in Step 25.
```
echo $SECURE_INGRESS_PORT
```

27. A Load Balancer is already created in AWS you can view it from EC2 Dashboard


28. Set the egress IP and ports for Gateway (Change the INGRESS_HOST to your LB DNS name)
```
export INGRESS_HOST=a27d197d321ba41fab46f3be35695b8a-362374880.us-west-1.elb.amazonaws.com

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

echo $GATEWAY_URL
```

29. Edit the Security group of EC2 instance created in Step 1. and add All Traffic (For Demo Purpose only) Inboud to your IP


30. Hit the URL from the below endpoint
```
echo "http://$GATEWAY_URL/productpage"
```
`Refresh the page to see the different versions of the applications deployed you will see the change in colour and no rating`


31. All Tools Installation (Kiali & Jaeger)
```
cd /root/istio-1.18.1/samples/addons
```
```
kubectl apply -f .
```

32. Do Port Forwarding for Kiali
```
kubectl port-forward --address 0.0.0.0 svc/kiali 9008:20001 -n istio-system
```

33. Now access the Kiali Dashboard using the below 

`<EC2-Instance-IP-From-Step1>:9008`


34. In Kiali dashboard -> Click on Graph on left pane -> Under Namespace click on Select all

     Now hit the endpoint from Step 30. multiple times


35. Do Port Forwarding for Jaeger
```
kubectl port-forward --address 0.0.0.0 svc/tracing 8008:80 -n istio-system
```

36. Now again hit the endpoint from Step 30. multiple times

     In Jaeger dashboard -> Select any of the Service (selecting istio-ingressgateway.istio-system) -> Click on Find Traces

     Click on any of the traces 


37. Cleaning up Resources used 
```
eksctl delete cluster --name eksdemo1 --region us-west-1
```

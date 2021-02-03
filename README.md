# Serverless PoC using kyma-lite

#### Shoot Configuration
- provider: GCP
- machine config: `n1-standard-2`
- Cluster Autoscaling: enabled to max 4 worker nodes min 1 worker node

#### Installing Kyma lite for Functions
using kyma-cli to install kyma-lite. refer the component file used as input for installation.
(filename: [serverless.yaml](serverless.yaml))
```shell
kyma install -c ./serverless.yaml
```


#### Installed components 
- Cluster Essentials
- istio
- xip-patch
- core
- serverless

#### Total Installation Time
- 7 mins 6 seconds

#### Installation Status
`SUCCESS`

#### Deploy a Function
```shell
kubectl apply -f - <<EOF
apiVersion: serverless.kyma-project.io/v1alpha1
kind: Function
metadata:
  name: test-fn
spec:
  buildResources:
    limits:
      cpu: 1100m
      memory: 1100Mi
    requests:
      cpu: 700m
      memory: 700Mi
  deps: "{ \n  \"name\": \"hilly-road\",\n  \"version\": \"1.0.0\",\n  \"dependencies\":
    {}\n}"
  maxReplicas: 1
  minReplicas: 1
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 50m
      memory: 64Mi
  runtime: nodejs12
  source: "module.exports = { \n  main: function (event, context) {\n    return \"Hello
    World!\";\n  }\n}"

---

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: vs-func
spec:
  gateways:
  - kyma-gateway.kyma-system.svc.cluster.local
  hosts:
  - test.aj-ser1.cf-kyma-k8.shoot.canary.k8s-hana.ondemand.com
  http:
   - match:
     - uri:
         regex: /.*
     route:
     - destination:
         host: test-fn
         port:
           number: 80
       weight: 100
EOF

```

#### Immediate Observation
- As soon as a function is deployed, cluster is scaled to 2 worker nodes, probably due to function build
  job resource utilization.
- Resource Utilisation on one running function under no load.
```shell
╰─ k top node                                                                                       ─╯
NAME                                                     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
shoot--cf-kyma-k8--aj-ser1-worker-jinug-z1-7dbf7-84mz6   205m         10%    1320Mi          21%
shoot--cf-kyma-k8--aj-ser1-worker-jinug-z1-7dbf7-zdl7l   336m         17%    1996Mi          32%
```


#### Errors Encountered
- kyma installation using cli usually retreives an `admin-user` secret, but due to
  missing `cluster-users` chart this secret was not created. (Can be mitigated by not using kyma-cli,
 or adding the secret in another chart, or install `cluster-users`.
- DNSEntry resource had to be created manually sometimes to make virtualservice host map to the istio-ingressgateway.



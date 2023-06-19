# Istio In Action


## :two: Chapter

```
istioctl x precheck
```
> Return
```

✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
  To get started, check out https://istio.io/latest/docs/setup/getting-started/
```


```
istioctl install --set profile=demo
```
> Return
```
This will install the Istio 1.13.3 demo profile with ["Istio core" "Istiod" "Ingress gateways" "Egress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed                                                                                                                       
✔ Istiod installed                                                                                                                           
✔ Egress gateways installed                                                                                                                  
✔ Ingress gateways installed                                                                                                                 
- Pruning removed resources                                                                                                                    Removed HorizontalPodAutoscaler:istio-system:ingressgateway.
  Removed PodDisruptionBudget:istio-system:ingressgateway.
  Removed Deployment:istio-system:ingressgateway.
  Removed Service:istio-system:ingressgateway.
  Removed ServiceAccount:istio-system:ingressgateway-service-account.
  Removed RoleBinding:istio-system:ingressgateway-sds.
  Removed Role:istio-system:ingressgateway-sds.
✔ Installation complete                                                                                                                      Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/pzWZpAvMVBecaQ9h9
```

## Create the Catalog Service

```
k apply -f catalog-service.yaml 
```

## example

```
k create namespace istioinaction
```

```
k config set-context $(kubectl config current-context) \
       --namespace=istioinaction
```

```
k config get-contexts
```
> Return
```
CURRENT   NAME                        CLUSTER                     AUTHINFO                    NAMESPACE
*         grappe-mineure.stp.uof.ca   grappe-mineure.stp.uof.ca   grappe-mineure.stp.uof.ca   istioinaction
          qa.moodle.k8s.local         qa.moodle.k8s.local         qa.moodle.k8s.local         
```

* from  ../book-source-code/services/catalog/kubernetes/catalog.yaml
```
istioctl kube-inject -f catalog.yaml
```
> Return
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalog
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: catalog
  name: catalog
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app: catalog
---
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: catalog
    version: v1
  name: catalog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalog
      version: v1
  strategy: {}
  template:
    metadata:
      annotations:
        kubectl.kubernetes.io/default-container: catalog
        kubectl.kubernetes.io/default-logs-container: catalog
        prometheus.io/path: /stats/prometheus
        prometheus.io/port: "15020"
        prometheus.io/scrape: "true"
        sidecar.istio.io/status: '{"initContainers":["istio-init"],"containers":["istio-proxy"],"volumes":["istio-envoy","istio-data","istio-podinfo","istio-token","istiod-ca-cert"],"imagePullSecrets":null,"revision":"default"}'
      creationTimestamp: null
      labels:
        app: catalog
        security.istio.io/tlsMode: istio
        service.istio.io/canonical-name: catalog
        service.istio.io/canonical-revision: v1
        version: v1
    spec:
      containers:
      - env:
        - name: KUBERNETES_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        image: istioinaction/catalog:latest
        imagePullPolicy: IfNotPresent
        name: catalog
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP
        resources: {}
        securityContext:
          privileged: false
      - args:
        - proxy
        - sidecar
        - --domain
        - $(POD_NAMESPACE).svc.cluster.local
        - --proxyLogLevel=warning
        - --proxyComponentLogLevel=misc:error
        - --log_output_level=default:info
        - --concurrency
        - "2"
        env:
        - name: JWT_POLICY
          value: third-party-jwt
        - name: PILOT_CERT_PROVIDER
          value: istiod
        - name: CA_ADDR
          value: istiod.istio-system.svc:15012
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: INSTANCE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: SERVICE_ACCOUNT
          valueFrom:
            fieldRef:
              fieldPath: spec.serviceAccountName
        - name: HOST_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: PROXY_CONFIG
          value: |
            {}
        - name: ISTIO_META_POD_PORTS
          value: |-
            [
                {"name":"http","containerPort":3000,"protocol":"TCP"}
            ]
        - name: ISTIO_META_APP_CONTAINERS
          value: catalog
        - name: ISTIO_META_CLUSTER_ID
          value: Kubernetes
        - name: ISTIO_META_INTERCEPTION_MODE
          value: REDIRECT
        - name: ISTIO_META_MESH_ID
          value: cluster.local
        - name: TRUST_DOMAIN
          value: cluster.local
        image: docker.io/istio/proxyv2:1.13.3
        name: istio-proxy
        ports:
        - containerPort: 15090
          name: http-envoy-prom
          protocol: TCP
        readinessProbe:
          failureThreshold: 30
          httpGet:
            path: /healthz/ready
            port: 15021
          initialDelaySeconds: 1
          periodSeconds: 2
          timeoutSeconds: 3
        resources:
          limits:
            cpu: "2"
            memory: 1Gi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: true
          runAsGroup: 1337
          runAsNonRoot: true
          runAsUser: 1337
        volumeMounts:
        - mountPath: /var/run/secrets/istio
          name: istiod-ca-cert
        - mountPath: /var/lib/istio/data
          name: istio-data
        - mountPath: /etc/istio/proxy
          name: istio-envoy
        - mountPath: /var/run/secrets/tokens
          name: istio-token
        - mountPath: /etc/istio/pod
          name: istio-podinfo
      initContainers:
      - args:
        - istio-iptables
        - -p
        - "15001"
        - -z
        - "15006"
        - -u
        - "1337"
        - -m
        - REDIRECT
        - -i
        - '*'
        - -x
        - ""
        - -b
        - '*'
        - -d
        - 15090,15021,15020
        image: docker.io/istio/proxyv2:1.13.3
        name: istio-init
        resources:
          limits:
            cpu: "2"
            memory: 1Gi
          requests:
            cpu: 10m
            memory: 40Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: false
          runAsGroup: 0
          runAsNonRoot: false
          runAsUser: 0
      serviceAccountName: catalog
      volumes:
      - emptyDir:
          medium: Memory
        name: istio-envoy
      - emptyDir: {}
        name: istio-data
      - downwardAPI:
          items:
          - fieldRef:
              fieldPath: metadata.labels
            path: labels
          - fieldRef:
              fieldPath: metadata.annotations
            path: annotations
        name: istio-podinfo
      - name: istio-token
        projected:
          sources:
          - serviceAccountToken:
              audience: istio-ca
              expirationSeconds: 43200
              path: istio-token
      - configMap:
          name: istio-ca-root-cert
        name: istiod-ca-cert
status: {}
---

```

```
k label namespace istioinaction istio-injection=enabled
```
> Return
```
namespace/istioinaction labeled
```

* from ../book-source-code/services/catalog/kubernetes/catalog.yaml
```
kubectl apply  -f catalog.yaml
```
> Return
```
serviceaccount/catalog created
service/catalog created
deployment.apps/catalog created
```

```
k get pod
```
> Return
```
NAME                     READY   STATUS    RESTARTS   AGE
catalog-6cf4b97d-b9gr8   2/2     Running   0          42s
```

- [ ] Test


```
k run -i -n default --rm --restart=Never dummy \
                    --image=curlimages/curl --command -- \
           sh -c 'curl -s http://catalog.istioinaction/items/1'
```
> Return
```json
{
  "id": 1,
  "color": "amber",
  "department": "Eyewear",
  "name": "Elinor Glasses",
  "price": "282.00"
}
pod "dummy" deleted
```

* from ../book-source-code/services/webapp/kubernetes/webapp.yaml
```
k apply -f webapp.yaml
```
> Return
```
serviceaccount/webapp created
service/webapp created
deployment.apps/webapp created
```

```
k get pods
```
> Return
```
NAME                     READY   STATUS    RESTARTS   AGE
catalog-6cf4b97d-b9gr8   2/2     Running   0          6m34s
webapp-7685bcb84-vbdvj   2/2     Running   0          7s
```

```
k run -i -n default --rm --restart=Never dummy \                     
          --image=curlimages/curl --command -- \
          sh -c 'curl -s http://webapp.istioinaction/api/catalog/items/1'
```
> Return
```json
{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"}
pod "dummy" deleted
```

```
k port-forward deploy/webapp 8080:8080
```
> Return
```
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```



# References

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.0 sh -
```
> Return
```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   101  100   101    0     0    620      0 --:--:-- --:--:-- --:--:--   647
100  4926  100  4926    0     0  20995      0 --:--:-- --:--:-- --:--:-- 20995

Downloading istio-1.13.0 from https://github.com/istio/istio/releases/download/1.13.0/istio-1.13.0-osx-arm64.tar.gz ...

Istio 1.13.0 Download Complete!

Istio has been successfully downloaded into the istio-1.13.0 folder on your system.

Next Steps:
See https://istio.io/latest/docs/setup/install/ to add Istio to your Kubernetes cluster.

To configure the istioctl client tool for your workstation,
add the /Users/u0000000021/Developer/config-grappe-mineure/prod/service-mesh/istio_in_action/istio-1.13.0/bin directory to your environment path variable with:
	 export PATH="$PATH:/Users/u0000000021/Developer/config-grappe-mineure/prod/service-mesh/istio_in_action/istio-1.13.0/bin"

Begin the Istio pre-installation check by running:
	 istioctl x precheck 

Need more information? Visit https://istio.io/latest/docs/setup/install/ 
```

```
istioctl verify-install
```
> Return
```
1 Istio control planes detected, checking --revision "default" only
✔ ClusterRole: istiod-istio-system.istio-system checked successfully
✔ ClusterRole: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istio-reader-service-account.istio-system checked successfully
✔ Role: istiod-istio-system.istio-system checked successfully
✔ RoleBinding: istiod-istio-system.istio-system checked successfully
✔ ServiceAccount: istiod-service-account.istio-system checked successfully
✔ CustomResourceDefinition: wasmplugins.extensions.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: destinationrules.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: envoyfilters.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: gateways.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: proxyconfigs.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: serviceentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: sidecars.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: virtualservices.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadentries.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: workloadgroups.networking.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: authorizationpolicies.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: peerauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: requestauthentications.security.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: telemetries.telemetry.istio.io.istio-system checked successfully
✔ CustomResourceDefinition: istiooperators.install.istio.io.istio-system checked successfully
✔ ClusterRole: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRole: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istiod-gateway-controller-istio-system.istio-system checked successfully
✔ ConfigMap: istio.istio-system checked successfully
✔ Deployment: istiod.istio-system checked successfully
✔ ConfigMap: istio-sidecar-injector.istio-system checked successfully
✔ MutatingWebhookConfiguration: istio-sidecar-injector.istio-system checked successfully
✔ PodDisruptionBudget: istiod.istio-system checked successfully
✔ ClusterRole: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ ClusterRoleBinding: istio-reader-clusterrole-istio-system.istio-system checked successfully
✔ Role: istiod.istio-system checked successfully
✔ RoleBinding: istiod.istio-system checked successfully
✔ Service: istiod.istio-system checked successfully
✔ ServiceAccount: istiod.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.11.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.11.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.12.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.12.istio-system checked successfully
✔ EnvoyFilter: stats-filter-1.13.istio-system checked successfully
✔ EnvoyFilter: tcp-stats-filter-1.13.istio-system checked successfully
✔ ValidatingWebhookConfiguration: istio-validator-istio-system.istio-system checked successfully
Checked 15 custom resource definitions
Checked 1 Istio Deployments
✔ Istio is installed and verified successfully
```

```
k apply -f istio-1.13.0/samples/addons 
```
> Return
```
serviceaccount/grafana created
configmap/grafana created
service/grafana created
deployment.apps/grafana created
configmap/istio-grafana-dashboards created
configmap/istio-services-grafana-dashboards created
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created
```

```
k get svc -n istio-system | awk '{print $1 " -> " $5}'
```
> Return
```
NAME -> PORT(S)
grafana -> 3000/TCP
ingressgateway -> 15021:31614/TCP,80:31717/TCP,443:31459/TCP
istio-egressgateway -> 80/TCP,443/TCP
istio-ingressgateway -> 15021:30257/TCP,80:31505/TCP,443:30884/TCP,31400:30876/TCP,15443:30281/TCP
istiod -> 15010/TCP,15012/TCP,443/TCP,15014/TCP
jaeger-collector -> 14268/TCP,14250/TCP,9411/TCP
kiali -> 20001/TCP,9090/TCP
prometheus -> 9090/TCP
tracing -> 80/TCP,16685/TCP
zipkin -> 9411/TCP
```


```
k port-forward services/kiali 20001:20001 -n istio-system
```

```
k port-forward services/grafana 3000:3000 -n istio-system
```

--

```
git clone https://github.com/istioinaction/book-source-code
```

```
kubectl apply -f ch2/ingress-gateway.yaml
```
> Return
```
gateway.networking.istio.io/coolstore-gateway created
virtualservice.networking.istio.io/webapp-virtualservice created
```

```
URL=$(kubectl -n istio-system get svc istio-ingressgateway \
-o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

```
curl $URL/api/catalog/items/1
```
> Return
```
{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"}
```

```
istioctl proxy-config routes \
> deploy/istio-ingressgateway.istio-system
```
> Return
```
NAME          DOMAINS     MATCH                  VIRTUAL SERVICE
http.8080     *           /*                     webapp-virtualservice.istioinaction
              *           /stats/prometheus*     
              *           /healthz/ready*       
```

```
istioctl proxy-config routes deploy/istio-ingressgateway.istio-system
```
> Return
```
NAME          DOMAINS     MATCH                  VIRTUAL SERVICE
http.8080     *           /*                     webapp-virtualservice.istioinaction
              *           /stats/prometheus*     
              *           /healthz/ready*        
```

```
k get gateway       
```
> Return
```
NAME                 AGE
outfitters-gateway   10m
```

```
k get virtualservice
```
> Return
```
NAME                    GATEWAYS                 HOSTS   AGE
webapp-virtualservice   ["outfitters-gateway"]   ["*"]   10m
```

```
k get pods                                                      
```
> Return
```
NAME                     READY   STATUS    RESTARTS   AGE
catalog-6cf4b97d-b9gr8   2/2     Running   0          3d14h
webapp-7685bcb84-vbdvj   2/2     Running   0          3d14h
```

```
istioctl proxy-config routes pod/catalog-6cf4b97d-b9gr8.istioinaction
```
> Return
```
NAME                                                          DOMAINS                                                                           MATCH                  VIRTUAL SERVICE
inbound|3000||                                                *                                                                                 /*                     
InboundPassthroughClusterIpv6                                 *                                                                                 /*                     
jaeger-collector.istio-system.svc.cluster.local:14268         *                                                                                 /*                     
InboundPassthroughClusterIpv4                                 *                                                                                 /*                     
ingressgateway.istio-system.svc.cluster.local:15021           *                                                                                 /*                     
grafana.istio-system.svc.cluster.local:3000                   *                                                                                 /*                     
istio-ingressgateway.istio-system.svc.cluster.local:15021     *                                                                                 /*                     
kube-dns.kube-system.svc.cluster.local:9153                   *                                                                                 /*                     
InboundPassthroughClusterIpv6                                 *                                                                                 /*                     
                                                              *                                                                                 /healthz/ready*        
inbound|3000||                                                *                                                                                 /*                     
jaeger-collector.istio-system.svc.cluster.local:14250         *                                                                                 /*                     
InboundPassthroughClusterIpv4                                 *                                                                                 /*                     
80                                                            catalog, catalog.istioinaction + 1 more...                                        /*                     
80                                                            catalog.prod.svc.cluster.local                                                    /*                     catalog-service.default
80                                                            catalog.prod.svc.cluster.local                                                    /*                     catalog-service.default
80                                                            ingress-nginx-ingress-controller-default-backend.bibliotheque, 100.64.200.215     /*                     
80                                                            ingress-nginx-ingress-controller.bibliotheque, 100.67.164.9                       /*                     
80                                                            ingressgateway.istio-system, 100.64.66.187                                        /*                     
80                                                            istio-egressgateway.istio-system, 100.64.183.163                                  /*                     
80                                                            istio-ingressgateway.istio-system, 100.68.226.202                                 /*                     
80                                                            moodle.stp, 100.64.244.204                                                        /*                     
80                                                            moodleingress-nginx-ingress-controller-default-backend.stp, 100.64.161.47         /*                     
80                                                            moodleingress-nginx-ingress-controller.stp, 100.66.180.69                         /*                     
80                                                            tracing.istio-system, 100.68.37.20                                                /*                     
80                                                            webapp, webapp.istioinaction + 1 more...                                          /*                     
80                                                            wordpress.bibliotheque, 100.64.173.205                                            /*                     
8383                                                          istio-operator.istio-operator, 100.65.19.10                                       /*                     
9090                                                          kiali.istio-system, 100.71.186.125                                                /*                     
9090                                                          prometheus.istio-system, 100.69.28.203                                            /*                     
9411                                                          jaeger-collector.istio-system, 100.71.142.81                                      /*                     
9411                                                          zipkin.istio-system, 100.69.136.75                                                /*                     
15010                                                         istiod.istio-system, 100.65.4.185                                                 /*                     
15014                                                         istiod.istio-system, 100.65.4.185                                                 /*                     
16685                                                         tracing.istio-system, 100.68.37.20                                                /*                     
20001                                                         kiali.istio-system, 100.71.186.125                                                /*                     
                                                              *                                                                                 /stats/prometheus*     
```

```
URL=$(kubectl -n istio-system get svc istio-ingressgateway \
              -o jsonpath='{.status.loadBalancer.ingress[0].hostname}') ; \
while true; do curl http://${URL}/api/catalog; sleep .5; done 
```

```
URL=$(kubectl -n istio-system get svc istio-ingressgateway \
              -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')    
```

```
curl -v http://${URL}/api/catalog
```
> Return
```
*   Trying 3.96.254.187:80...
* Connected to aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com (3.96.254.187) port 80 (#0)
> GET /api/catalog HTTP/1.1
> Host: aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com
> User-Agent: curl/7.79.1
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 500 Internal Server Error
< date: Thu, 05 May 2022 20:40:16 GMT
< content-length: 29
< content-type: text/plain; charset=utf-8
< x-envoy-upstream-service-time: 87
< server: istio-envoy
< 
* Connection #0 to host aed1f19c407c74379bda3504355fc3c1-616782180.ca-central-1.elb.amazonaws.com left intact
error calling Catalog service
```

```
kubectl delete deployment,svc,virtualservice,destinationrule --all -n istioinaction 
```
> Return
```
deployment.apps "catalog" deleted
deployment.apps "catalog-v2" deleted
deployment.apps "webapp" deleted
service "catalog" deleted
service "webapp" deleted
virtualservice.networking.istio.io "catalog" deleted
virtualservice.networking.istio.io "webapp-virtualservice" deleted
destinationrule.networking.istio.io "catalog" deleted
```


# References

- [ ] [Deploy a Custom Ingress Gateway Using Cert-Manager ](https://istio.io/v1.3/blog/2019/custom-ingress-gateway/)

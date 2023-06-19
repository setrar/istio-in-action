# Istio

## :gear: istio version

```
istioctl version     
```
> Return
```
no running Istio pods in "istio-system"
1.13.3
```

## :cl:  Istio Profile

- [ ] List

```
istioctl profile list
```
> Return
```
Istio configuration profiles:
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote
```

- [ ] Dump

```
istioctl profile dump demo
```
> Return
```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: true
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
        service:
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          - name: http2
            port: 80
            targetPort: 8080
          - name: https
            port: 443
            targetPort: 8443
          - name: tcp
            port: 31400
            targetPort: 31400
          - name: tls
            port: 15443
            targetPort: 15443
      name: istio-ingressgateway
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
      k8s:
        env:
        - name: PILOT_TRACE_SAMPLING
          value: "100"
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
  hub: docker.io/istio
  meshConfig:
    accessLogFile: /dev/stdout
    defaultConfig:
      proxyMetadata: {}
    enablePrometheusMerge: true
    extensionProviders:
    - envoyOtelAls:
        port: 4317
        service: otel-collector.istio-system.svc.cluster.local
      name: otel
  profile: demo
  tag: 1.13.3
  values:
    base:
      enableCRDTemplates: false
      validationURL: ""
    defaultRevision: ""
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-egressgateway
        secretVolumes:
        - mountPath: /etc/istio/egressgateway-certs
          name: egressgateway-certs
          secretName: istio-egressgateway-certs
        - mountPath: /etc/istio/egressgateway-ca-certs
          name: egressgateway-ca-certs
          secretName: istio-egressgateway-ca-certs
        type: ClusterIP
      istio-ingressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-ingressgateway
        secretVolumes:
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          secretName: istio-ingressgateway-certs
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          secretName: istio-ingressgateway-ca-certs
        type: LoadBalancer
    global:
      configValidation: true
      defaultNodeSelector: {}
      defaultPodDisruptionBudget:
        enabled: true
      defaultResources:
        requests:
          cpu: 10m
      imagePullPolicy: ""
      imagePullSecrets: []
      istioNamespace: istio-system
      istiod:
        enableAnalysis: false
      jwtPolicy: third-party-jwt
      logAsJson: false
      logging:
        level: default:info
      meshNetworks: {}
      mountMtlsCerts: false
      multiCluster:
        clusterName: ""
        enabled: false
      network: ""
      omitSidecarInjectorConfigMap: false
      oneNamespace: false
      operatorManageWebhooks: false
      pilotCertProvider: istiod
      priorityClassName: ""
      proxy:
        autoInject: enabled
        clusterDomain: cluster.local
        componentLogLevel: misc:error
        enableCoreDump: false
        excludeIPRanges: ""
        excludeInboundPorts: ""
        excludeOutboundPorts: ""
        image: proxyv2
        includeIPRanges: '*'
        logLevel: warning
        privileged: false
        readinessFailureThreshold: 30
        readinessInitialDelaySeconds: 1
        readinessPeriodSeconds: 2
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 10m
            memory: 40Mi
        statusPort: 15020
        tracer: zipkin
      proxy_init:
        image: proxyv2
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 10m
            memory: 10Mi
      sds:
        token:
          aud: istio-ca
      sts:
        servicePort: 0
      tracer:
        datadog: {}
        lightstep: {}
        stackdriver: {}
        zipkin: {}
      useMCP: false
    istiodRemote:
      injectionURL: ""
    pilot:
      autoscaleEnabled: false
      autoscaleMax: 5
      autoscaleMin: 1
      configMap: true
      cpu:
        targetAverageUtilization: 80
      enableProtocolSniffingForInbound: true
      enableProtocolSniffingForOutbound: true
      env: {}
      image: pilot
      keepaliveMaxServerConnectionAge: 30m
      nodeSelector: {}
      podLabels: {}
      replicaCount: 1
      traceSampling: 1
    telemetry:
      enabled: true
      v2:
        enabled: true
        metadataExchange:
          wasmEnabled: false
        prometheus:
          enabled: true
          wasmEnabled: false
        stackdriver:
          configOverride: {}
          enabled: false
          logging: false
          monitoring: false
          topology: false
```

## :a: Installing and customizing Istio using `istioctl`

- [ ] Install the Contro Plane Demo Profile [without gateways](appendices/demo-profile-without-gateways.yaml )

```
istioctl install --filename appendices/demo-profile-without-gateways.yaml 
```
> Return
```
This will install the Istio 1.13.3 demo profile with ["Istio core" "Istiod"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed                                                                                                                       
✔ Istiod installed                                                                                                                           
✔ Installation complete                                                                                                                      Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/pzWZpAvMVBecaQ9h9
```

- [ ] Check `CRDs` are created

```
k get crd -n istio-system
```
> Return
```
NAME                                       CREATED AT
authorizationpolicies.security.istio.io    2022-05-01T22:05:28Z
certificaterequests.cert-manager.io        2022-04-21T17:17:25Z
certificates.cert-manager.io               2022-04-21T17:17:26Z
challenges.acme.cert-manager.io            2022-04-21T17:17:26Z
clusterissuers.cert-manager.io             2022-04-21T17:17:26Z
destinationrules.networking.istio.io       2022-05-01T22:05:28Z
envoyfilters.networking.istio.io           2022-05-01T22:05:28Z
gateways.networking.istio.io               2022-05-01T22:05:28Z
issuers.cert-manager.io                    2022-04-21T17:17:26Z
istiooperators.install.istio.io            2022-05-01T22:05:28Z
orders.acme.cert-manager.io                2022-04-21T17:17:26Z
peerauthentications.security.istio.io      2022-05-01T22:05:29Z
proxyconfigs.networking.istio.io           2022-05-01T22:05:29Z
requestauthentications.security.istio.io   2022-05-01T22:05:29Z
serviceentries.networking.istio.io         2022-05-01T22:05:29Z
sidecars.networking.istio.io               2022-05-01T22:05:29Z
telemetries.telemetry.istio.io             2022-05-01T22:05:29Z
virtualservices.networking.istio.io        2022-05-01T22:05:29Z
wasmplugins.extensions.istio.io            2022-05-01T22:05:29Z
workloadentries.networking.istio.io        2022-05-01T22:05:29Z
workloadgroups.networking.istio.io         2022-05-01T22:05:29Z
```

- [ ] Install the data plane Demo Profile [gateways](appendices/ingress-gateway.yaml )

```
istioctl install --filename appendices/ingress-gateway.yaml 
```
> Return
```
This will install the Istio 1.13.3 empty profile into the cluster. Proceed? (y/N) y
✔ Ingress gateways installed                                                                                                                 
✔ Installation complete                                                                                                                      Making this installation the default for injection and validation.

Thank you for installing Istio 1.13.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/pzWZpAvMVBecaQ9h9
```

- [ ] Check Load Balancer is created

```
kubectl get svc ingressgateway \
    --namespace istio-system \
    --output jsonpath="{.status.loadBalancer.ingress[0].hostname}"; echo
```
> Return
```
a2610b09dd7f74a989a04ae34fd206f2-1569156610.ca-central-1.elb.amazonaws.com
```

## :b: Installing and customizing Istio with the `istio-operator`

- [ ] Installing the `istio-operator`

```
istioctl operator init
```
> Return
```
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.13.3
Operator controller will watch namespaces: istio-system
✔ Istio operator installed                                                                                                                   
✔ Installation complete
```

- [ ] List the associated `CRDs`

* namespace `istio-operator`

```
k get crd -n istio-operator
```
> Return
```
NAME                                  CREATED AT
certificaterequests.cert-manager.io   2022-04-21T17:17:25Z
certificates.cert-manager.io          2022-04-21T17:17:26Z
challenges.acme.cert-manager.io       2022-04-21T17:17:26Z
clusterissuers.cert-manager.io        2022-04-21T17:17:26Z
issuers.cert-manager.io               2022-04-21T17:17:26Z
istiooperators.install.istio.io       2022-05-01T22:52:05Z
orders.acme.cert-manager.io           2022-04-21T17:17:26Z
```

* namespace `istio-system`

```
k get crd -n istio-system  
```
> Return
```
NAME                                  CREATED AT
certificaterequests.cert-manager.io   2022-04-21T17:17:25Z
certificates.cert-manager.io          2022-04-21T17:17:26Z
challenges.acme.cert-manager.io       2022-04-21T17:17:26Z
clusterissuers.cert-manager.io        2022-04-21T17:17:26Z
issuers.cert-manager.io               2022-04-21T17:17:26Z
istiooperators.install.istio.io       2022-05-01T22:52:05Z
orders.acme.cert-manager.io           2022-04-21T17:17:26Z
```

- [ ] Install the Contro Plane Demo Profile [without gateways](appendices/demo-profile-without-gateways.yaml )

:bulb: Note: `kubectl` is used rather than `istioctl`

```
kubectl apply --filename appendices/demo-profile-without-gateways.yaml -n istio-system
```
> Return
```
istiooperator.install.istio.io/control-plane created
```

- [ ] Install the data plane Demo Profile [gateways](appendices/ingress-gateway.yaml )


```
kubectl apply --filename  appendices/ingress-gateway.yaml -n istio-system
```
> Return
```
istiooperator.install.istio.io/ingress-gateway created
```

- [ ] Check Load Balancer is created

```
kubectl get svc ingressgateway \
    --namespace istio-system \
    --output jsonpath="{.status.loadBalancer.ingress[0].hostname}"; echo
```
> Return
```
ad0cd2b6625154807b0ace3e6a30a11b-217741452.ca-central-1.elb.amazonaws.com
```

## :x: [Uninstall istio](https://istio.io/latest/docs/setup/install/istioctl/#uninstall-istio)

- [ ] Delete `istio-system` namespace

```
kubectl delete namespace istio-system
```
> Return
```
namespace "istio-system" deleted
```

- [ ] Purge `istio` artifacts

```
istioctl x uninstall --purge
```
> Return
```
failed to get proxy infos: unable to find any Istiod instances
All Istio resources will be pruned from the cluster
Proceed? (y/N) y
  Removed MutatingWebhookConfiguration::istio-revision-tag-default.
  Removed MutatingWebhookConfiguration::istio-sidecar-injector.
  Removed ValidatingWebhookConfiguration::istio-validator-istio-system.
  Removed ValidatingWebhookConfiguration::istiod-default-validator.
  Removed ClusterRole::istio-reader-clusterrole-istio-system.
  Removed ClusterRole::istio-reader-istio-system.
  Removed ClusterRole::istiod-clusterrole-istio-system.
  Removed ClusterRole::istiod-gateway-controller-istio-system.
  Removed ClusterRole::istiod-istio-system.
  Removed ClusterRoleBinding::istio-reader-clusterrole-istio-system.
  Removed ClusterRoleBinding::istio-reader-istio-system.
  Removed ClusterRoleBinding::istiod-clusterrole-istio-system.
  Removed ClusterRoleBinding::istiod-gateway-controller-istio-system.
  Removed ClusterRoleBinding::istiod-istio-system.
  Removed CustomResourceDefinition::authorizationpolicies.security.istio.io.
  Removed CustomResourceDefinition::destinationrules.networking.istio.io.
  Removed CustomResourceDefinition::envoyfilters.networking.istio.io.
  Removed CustomResourceDefinition::gateways.networking.istio.io.
  Removed CustomResourceDefinition::istiooperators.install.istio.io.
  Removed CustomResourceDefinition::peerauthentications.security.istio.io.
  Removed CustomResourceDefinition::proxyconfigs.networking.istio.io.
  Removed CustomResourceDefinition::requestauthentications.security.istio.io.
  Removed CustomResourceDefinition::serviceentries.networking.istio.io.
  Removed CustomResourceDefinition::sidecars.networking.istio.io.
  Removed CustomResourceDefinition::telemetries.telemetry.istio.io.
  Removed CustomResourceDefinition::virtualservices.networking.istio.io.
  Removed CustomResourceDefinition::wasmplugins.extensions.istio.io.
  Removed CustomResourceDefinition::workloadentries.networking.istio.io.
  Removed CustomResourceDefinition::workloadgroups.networking.istio.io.
✔ Uninstall complete                          
```



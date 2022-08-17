## Lab 13 - Deploy sleep containers for zero-trust lab <a name="Lab-13"></a>
If you do not have ephemeral containers feature flag turned on, we can replace the functionality with the sleep demo app

Run the following commands to deploy the sleep app on `cluster1` twice

The first version will be called `sleep-not-in-mesh` and won't have the sidecar injected (because we don't label the namespace).
```bash
kubectl --context ${CLUSTER1} create ns sleep

kubectl --context ${CLUSTER1} apply -n sleep -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sleep-not-in-mesh
  namespace: sleep
---
apiVersion: v1
kind: Service
metadata:
  name: sleep-not-in-mesh
  namespace: sleep
  labels:
    app: sleep-not-in-mesh
    service: sleep-not-in-mesh
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: sleep-not-in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep-not-in-mesh
  namespace: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep-not-in-mesh
  template:
    metadata:
      labels:
        app: sleep-not-in-mesh
    spec:
      terminationGracePeriodSeconds: 0
      serviceAccountName: sleep-not-in-mesh
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /etc/sleep-not-in-mesh/tls
          name: secret-volume
      volumes:
      - name: secret-volume
        secret:
          secretName: sleep-not-in-mesh-secret
          optional: true
EOF
```

The second version will be called sleep-in-mesh and will have the sidecar injected (because of the label istio.io/rev in the Pod template).
```bash
kubectl --context ${CLUSTER1} apply -n sleep -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sleep-in-mesh
  namespace: sleep
---
apiVersion: v1
kind: Service
metadata:
  name: sleep-in-mesh
  namespace: sleep
  labels:
    app: sleep-in-mesh
    service: sleep-in-mesh
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: sleep-in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep-in-mesh
  namespace: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep-in-mesh
  template:
    metadata:
      labels:
        app: sleep-in-mesh
        istio.io/rev: 1-13
    spec:
      terminationGracePeriodSeconds: 0
      serviceAccountName: sleep-in-mesh
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /etc/sleep-in-mesh/tls
          name: secret-volume
      volumes:
      - name: secret-volume
        secret:
          secretName: sleep-in-mesh-secret
          optional: true
EOF
```


## Lab 14 - Zero trust <a name="Lab-14"></a>

In the previous step, we federated multiple meshes and established a shared root CA for a shared identity domain.

All the communications between Pods in the mesh are now encrypted by default, but:

- communications between services that are in the mesh and others which aren't in the mesh are still allowed and not encrypted
- all the services can talk together

Let's validate this.

Run the following commands to initiate a communication from a service which isn't in the mesh to a service which is in the mesh:

With Sleep App:
```
kubectl --context ${CLUSTER1} exec -it -n sleep deploy/sleep-not-in-mesh -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

With Ephemeral Containers:
```
pod=$(kubectl --context ${CLUSTER1} -n httpbin get pods -l app=not-in-mesh -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${CLUSTER1} -n httpbin debug -i -q ${pod} --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

You should get a `200` response code which confirm that the communication is currently allowed.

Run the following commands to initiate a communication from a service which is in the mesh to another service which is in the mesh:

With Sleep App:
```
kubectl --context ${CLUSTER1} exec -it -n sleep deploy/sleep-in-mesh -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

With Ephemeral Containers:
```
pod=$(kubectl --context ${CLUSTER1} -n httpbin get pods -l app=in-mesh -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${CLUSTER1} -n httpbin debug -i -q ${pod} --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

You should get a `200` response code again.

To enfore a zero trust policy, it shouldn't be the case.

If you run the commands below, you will find that no `PeerAuthentication`, `AuthorizationPolicy`, or `Sidecars` have been configured to enforce zero trust.
```
kubectl get PeerAuthentication -A --context ${CLUSTER1}
kubectl get AuthorizationPolicy -A --context ${CLUSTER1}
kubectl get Sidecars -A --context ${CLUSTER1}
```

When running these commands you should see an output that says `No resources found`. This is expected

Now we'll leverage the Gloo Mesh workspaces to get to a state where:

- communications between services which are in the mesh and others which aren't in the mesh aren't allowed anymore
- communications between services in the mesh are allowed only when services are in the same workspace or when their workspaces have import/export rules.

The Bookinfo team must update its `WorkspaceSettings` Kubernetes object to enable service isolation.

```bash
kubectl apply --context ${CLUSTER1} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: bookinfo
  namespace: bookinfo-frontends
spec:
  importFrom:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
  exportTo:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
      labels:
        app: productpage
    - kind: SERVICE
      labels:
        app: reviews
    - kind: ALL
      labels:
        expose: "true"
  options:
    serviceIsolation:
      enabled: true
      trimProxyConfig: true
EOF
```

When service isolation is enabled, Gloo Mesh creates the corresponding Istio `AuthorizationPolicy` and `PeerAuthentication` objects, as well as configures the `Sidecar` objects to enforce zero trust.

When `trimProxyConfig` is set to `true`, Gloo Mesh also creates the corresponding Istio `Sidecar` objects to program the sidecar proxies to only know how to talk to the authorized services.

To validate this, run the commands below:
```
kubectl get PeerAuthentication -A --context ${CLUSTER1}
kubectl get AuthorizationPolicy -A --context ${CLUSTER1}
kubectl get Sidecars -A --context ${CLUSTER1}
```

To dig in deeper, you can run a `kubectl get <resource> -n <namespace> -o yaml` to see more details on what Gloo Mesh is doing under the hood

For example:
```
kubectl get AuthorizationPolicy -n bookinfo-frontends settings-productpage-9080-bookinfo -o yaml
```

Will yield the AuthorizationPolicy that was automatically generated by Gloo Mesh when workspace isolation was turned on:
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  creationTimestamp: "2022-06-14T20:40:44Z"
  generation: 1
  labels:
    agent.gloo.solo.io: gloo-mesh
    cluster.multicluster.solo.io: ""
    context.mesh.gloo.solo.io/cluster: cluster1
    context.mesh.gloo.solo.io/namespace: bookinfo-frontends
    context.mesh.gloo.solo.io/workspace: bookinfo
    gloo.solo.io/parent_cluster: cluster1
    gloo.solo.io/parent_group: ""
    gloo.solo.io/parent_kind: Service
    gloo.solo.io/parent_name: productpage
    gloo.solo.io/parent_namespace: bookinfo-frontends
    gloo.solo.io/parent_version: v1
    owner.gloo.solo.io/name: gloo-mesh
    reconciler.mesh.gloo.solo.io/name: translator
    relay.solo.io/cluster: cluster1
  name: settings-productpage-9080-bookinfo
  namespace: bookinfo-frontends
  resourceVersion: "9799"
  uid: 23499fc1-c1fe-49b5-8a84-ca454214d99d
spec:
  rules:
  - from:
    - source:
        principals:
        - cluster1/ns/bookinfo-backends/sa/bookinfo-details
        - cluster1/ns/bookinfo-backends/sa/bookinfo-ratings
        - cluster1/ns/bookinfo-backends/sa/bookinfo-reviews
        - cluster1/ns/bookinfo-backends/sa/default
        - cluster1/ns/bookinfo-frontends/sa/bookinfo-productpage
        - cluster1/ns/bookinfo-frontends/sa/default
        - cluster2/ns/bookinfo-backends/sa/bookinfo-details
        - cluster2/ns/bookinfo-backends/sa/bookinfo-ratings
        - cluster2/ns/bookinfo-backends/sa/bookinfo-reviews
        - cluster2/ns/bookinfo-backends/sa/default
        - cluster2/ns/bookinfo-frontends/sa/bookinfo-productpage
        - cluster2/ns/bookinfo-frontends/sa/default
    - source:
        principals:
        - cluster1/ns/gloo-mesh-addons/sa/default
        - cluster1/ns/gloo-mesh-addons/sa/ext-auth-service
        - cluster1/ns/gloo-mesh-addons/sa/rate-limiter
        - cluster1/ns/istio-gateways/sa/default
        - cluster1/ns/istio-gateways/sa/istio-eastwestgateway
        - cluster1/ns/istio-gateways/sa/istio-ingressgateway
        - cluster2/ns/gloo-mesh-addons/sa/default
        - cluster2/ns/gloo-mesh-addons/sa/ext-auth-service
        - cluster2/ns/gloo-mesh-addons/sa/rate-limiter
        - cluster2/ns/istio-gateways/sa/default
        - cluster2/ns/istio-gateways/sa/istio-eastwestgateway
        - cluster2/ns/istio-gateways/sa/istio-ingressgateway
    to:
    - operation:
        ports:
        - "9080"
  selector:
    matchLabels:
      app: productpage
```

If you refresh the browser, you'll see that the bookinfo application is still exposed and working correctly.

Run the following commands to initiate a communication from a service which isn't in the mesh to a service which is in the mesh:

With Sleep App:
```
kubectl --context ${CLUSTER1} exec -it -n sleep deploy/sleep-not-in-mesh -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

With Ephemeral Containers:
```
pod=$(kubectl --context ${CLUSTER1} -n httpbin get pods -l app=not-in-mesh -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${CLUSTER1} -n httpbin debug -i -q ${pod} --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

You should get a `000` response code which means that the communication can't be established.

Run the following commands to initiate a communication from a service which is in the mesh to another service which is in the mesh:

With Sleep App:
```
kubectl --context ${CLUSTER1} exec -it -n sleep deploy/sleep-in-mesh -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

With Ephemeral Containers:
```
pod=$(kubectl --context ${CLUSTER1} -n httpbin get pods -l app=in-mesh -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${CLUSTER1} -n httpbin debug -i -q ${pod} --image=curlimages/curl -- curl -s -o /dev/null -w "%{http_code}" http://reviews.bookinfo-backends.svc.cluster.local:9080/reviews/0
```

You should get a `403` response code which means that the sidecar proxy of the `reviews` service doesn't allow the request.

You've achieved zero trust with nearly no effort.
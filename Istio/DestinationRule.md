##### Defining subsets of application. 
This can be used when we want separate traffic:
```
apiVersion: networking.istio.io/v1alpha3 
kind: DestinationRule 
metadata: 
	name: catalog 
spec: 
	host: catalog 
	subsets: 
	- name: version-v1 
		labels: version: v1 
	- name: version-v2 
		labels: version: v2
```

##### Client-side load balancing
###### Round robin
```
apiVersion: networking.istio.io/v1beta1 
kind: DestinationRule 
metadata: 
	name: simple-backend-dr 
spec: 
	host: simple-backend 
	trafficPolicy: 
		loadBalancer: 
			simple: ROUND_ROBIN
```

##### Circuit breaking
```
apiVersion: networking.istio.io/v1beta1 
kind: DestinationRule 
metadata: 
	name: simple-backend-dr 
spec: 
	host: simple-backend.istioinaction.svc.cluster.local 
	trafficPolicy: 
		connectionPool: 
			tcp: 
				maxConnections: 1 (Total number of connections)
			http: 
				http1MaxPendingRequests: 1 (Queued requests)
				maxRequestsPerConnection: 1 (Requests per connection)
				maxRetries: 1 
				http2MaxRequests: 1 (Maximum concurrent requests to all hosts)
```
 - maxConnections — The threshold at which we report a connection overflow. The Istio proxy (Envoy) uses connections to service requests up to an upper bound defined in this setting. In reality, we can expect the maximum number of connections to be one per endpoint in the load-balancing pool plus the value of this setting. Any time we go over this value, Envoy will report it in its metrics.  
 - http1MaxPendingRequests — The allowable number of requests that are pending and don’t have a connection to use.  
 - http2MaxRequests — This setting is unfortunately misnamed in Istio. Under the covers, it controls the maximum number of parallel requests across all end-points/hosts in a cluster regardless of HTTP2 or HTTP1.1 (see https:// github.com/istio/istio/issues/27473).
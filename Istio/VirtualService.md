In Istio, a VirtualService resource defines how a client talks to a specific service through its fully qualified domain name, which versions of a service are available, and other routing properties (like retries and request timeouts).

##### Split traffic between two versions of application by request header
To make it works it's necessary a [[DestinationRule#Defining subsets of application.]] in which these versions are declared.
```
apiVersion: networking.istio.io/v1alpha3 
kind: VirtualService 
metadata: 
	name: catalog-vs 
spec: 
	hosts: 
	- catalog 
	http: 
	- match: 
		- headers: 
			x-istio-cohort: 
				exact: "internal" 
		route: 
			- destination: 
				host: catalog 
				subset: version-v2 
		- route: 
			- destination: 
				host: catalog 
				subset: version-v1
```

##### Split traffic between two versions of application by weight
To make it works it's necessary a [[DestinationRule#Defining subsets of application.]] in which these versions are declared.
```
apiVersion: networking.istio.io/v1alpha3 
kind: VirtualService 
metadata: 
	name: catalog-vs 
spec: 
	hosts: 
	- catalog
	http: 
		- route: 
			- destination: 
				host: catalog 
				subset: version-v1
			  weight: 90
			- destination: 
				host: catalog 
				subset: version-v2
			  weight: 10
```

##### Traffic mirroring to another version of application
To make it works it's necessary a [[DestinationRule#Defining subsets of application.]] in which these versions are declared. Mirroring is done in fire-and-forget manner in which a copy of the request is created and sent to the mirrored cluster. This mirrored request cannot affect the real request because the Istio proxy that does the mirroring ignores any responses (success/failure) from the mirrored cluster.
```
apiVersion: networking.istio.io/v1alpha3 
kind: VirtualService 
metadata: 
	name: catalog-vs 
spec: 
	hosts: 
	- catalog
	http: 
		- route: 
			- destination: 
				host: catalog 
				subset: version-v1
			  weight: 100
			mirror: 
				host: catalog 
				subset: version-v2
```

##### Timeouts
```
apiVersion: networking.istio.io/v1alpha3 
kind: VirtualService 
metadata: 
	name: simple-backend-vs 
spec: 
	hosts: 
	- simple-backend 
	http: 
	- route: 
	- destination: 
		host: simple-backend 
	timeout: 0.5s
```

##### Retries
Istio has retries enabled by default and will retry up to two times
```
apiVersion: networking.istio.io/v1alpha3 
kind: VirtualService 
metadata: 
	name: simple-backend-vs 
spec: 
	hosts: 
	- simple-backend 
	http: 
	- route: 
	- destination: 
		host: simple-backend 
	retries:
		attempts: 3
		retryOn: gateway-error,connect-failure,retriable-4xx (Errors to retry)
		perTryTimeout: 300ms 
		retryRemoteLocalities: true (Whether to retry endpoints in other localities)
```

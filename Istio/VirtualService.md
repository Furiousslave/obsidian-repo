In Istio, a VirtualService resource defines how a client talks to a specific service through its fully qualified domain name, which versions of a service are available, and other routing properties (like retries and request timeouts).

###### Routing traffic between two versions of application
To make it works it's necessary a [[DestinationRule#Defining subsets of application.]] in which these versions are declared.
```
apiVersion: networking.istio.io/v1alpha3 
kind: VirtualService 
metadata: 
	name: catalog-vs-from-gw 
spec: 
	hosts: 
	- "catalog.istioinaction.io" 
	gateways: 
	- catalog-gateway 
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


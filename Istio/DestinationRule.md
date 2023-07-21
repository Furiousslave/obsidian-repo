###### Defining subsets of application. 
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

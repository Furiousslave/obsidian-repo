Using ServiceEntry we are adding external service host into Istio's service registry. This is useful when outbound traffic allowed only to hosts in the Istio's service registry. It can be done by next command:
```
istioctl install --set profile=demo \ --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```
ServiceEntry example:
```
apiVersion: networking.istio.io/v1alpha3 
kind: ServiceEntry 
metadata: 
	name: jsonplaceholder 
spec: 
	hosts: 
	- jsonplaceholder.typicode.com 
	ports: 
	- number: 80 
	  name: http 
	  protocol: HTTP 
	resolution: DNS 
	location: MESH_EXTERNAL
```
Jks and TrustStore are standart secure storages for private keys, certifcates for server approach and client approach in Java applications.

##### JKS(serverr)
JKS is secure storage that stores private keys and certificates(for server).
Creating a JSK with a self-signed certificate:
```
keytool -genkeypair -alias myserver -keyalg RSA -keysize 2048 -keystore keystore.jks
```

##### TrustStore(client)
TrustStore is secure storage that stores trusted certifcates(usually Root?) 
Creating a TrustStore with imported Trusted Party Certificate:
```
keytool -import -alias trustedca -file trustedca.crt -keystore truststore.jks
```

JKS and TrustStore can be stored in one .jks file like this:
1. First create separate JKS and TrustStore files: 
```
keytool -genkeypair -alias myserver -keyalg RSA -keysize 2048 -keystore keystore.jks
keytool -import -alias trustedca -file trustedca.crt -keystore truststore.jks
```
2. Then combine them using the keytool  and define a new password for the combined file:
```
keytool -importkeystore -srckeystore keystore.jks -srcstorepass <keystore_password> -destkeystore combined.jks -deststorepass <combined_password>

keytool -importkeystore -srckeystore truststore.jks -srcstorepass <truststore_password> -destkeystore combined.jks -deststorepass <combined_password>
```
# Mutual TLS with Nginx Ingress Controleer
Based on instructions from: https://github.com/kubernetes/ingress-nginx/tree/master/docs/examples/auth/client-certs, https://medium.com/@awkwardferny/configuring-certificate-based-mutual-authentication-with-kubernetes-ingress-nginx-20e7e38fdfca

How to enable Mutual TLS with Nginx Ingress Controller.

Tested on AWS EKS cluster.

# Create Certificates

To demonstrate mutual TLS we can use self generated certificates.

Generate sample CA certificate:
```
openssl req -x509 -sha256 -newkey rsa:4096 -keyout ca.key -out ca.crt -days 356 -nodes -subj '/CN=Demo Certificate Authority'
```

Generate backend key and certificate signing request:
```
openssl req -new -newkey rsa:4096 -keyout backend.key -out backend.csr -nodes -subj '/CN=nginxhello.com'
```

Generate backend certificate by signing with previously created CA certificate:
```
openssl x509 -req -sha256 -days 365 -in backend.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out backend.crt
```

Repeat this process to generate client certificates:
```
openssl req -new -newkey rsa:4096 -keyout client.key -out client.csr -nodes -subj '/CN=Demo Client'
openssl x509 -req -sha256 -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 02 -out client.crt
```

# Create Kubernetes Secrets

We need to store our generated CA and backend certifcates in Kubernetes Secret objects so that they can be referenced by the Ingress.

Create and store certificates in kubernetes secrets:
```
kubectl create secret generic demo-ca --from-file=ca.crt=ca.crt
kubectl create secret generic demo-tls --from-file=tls.crt=backend.crt --from-file=tls.key=backend.key
```

Note: If the Kubernetes Secret name is changed when it is created, the same change must be made to the relevant references in the ing.yaml file.

# Create Kubernetes Resources

We can now deploy our sample webapp. Note the annotations in the ing.yaml file that enable Mutual TLS:
```
...
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "default/demo-ca"
...
```

Here, these annotations enable client verification(mutual TLS) and specify the secret containing our generated CA certificate. Our backend certificate is specified in the tls section:
```
...
tls:
  - hosts:
    - nginxhello.com
    secretName: demo-tls
...    
```

Create the deployment:
```
kubectl create -f deploy.yaml
```

Create the service:
```
kubectl create -f svc.yaml
```

Create the ingress:
```
kubectl create -f ing.yaml
```

# Testing

Find your Nginx Ingress Controller's External IP by the command:
```
kubectl get svc --namespace=ingress-nginx
```

Note: If you are running kubernetes on AWS, you will see a load balancer DNS instead of an IP, to get the IP you need run:
```
nslookup <Load Balancer DNS>

```
Configure the DNS names by appending your /etc/hosts file with:
```
<Nginx Ingress External IP>  nginxhello.com
```

Run a curl command without the client cetificates:
```
curl https://nginxhello.com/ -k
```

An output with below content should be returned:
```
<html>
<head><title>400 No required SSL certificate was sent</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>No required SSL certificate was sent</center>
<hr><center>nginx/1.15.8</center>
</body>
</html>
```

Now run a curl command with the client cetificates: 
```
curl https://nginxhello.com/ -k --cert client.crt --key client.key
```

An output with content similar to this should be returned:
```
Server address: <Your server address>
Server name: <Your server name>
Date: <Current Date>
URI: /
Request ID: <Your request ID>

```


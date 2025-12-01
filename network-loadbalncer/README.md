
🚀 Completed a full end-to-end mTLS (Mutual TLS) implementation on AWS — using a custom Root CA, Intermediate CA, client certificates, and NLB (TCP passthrough).

This setup is commonly used in banking, fintech, payment systems, and secure B2B APIs.
The goal: ensure both server and client authenticate each other over an encrypted channel.

🔐 What I built:

Created a full PKI chain (Root CA → Intermediate CA)

Generated server + client certificates

Configured nginx to enforce mTLS

Deployed the service behind an AWS Network Load Balancer (L4)

Verified that TLS remains end-to-end and client certificates are validated by the server

Tested successful & failed handshake scenarios

💡 Why NLB (not ALB)?

NLB supports TLS passthrough, so encrypted traffic reaches the EC2 instance directly — allowing the server to validate client certificates.
(This is not possible with ALB, which terminates TLS.)

This was a great hands-on experience and gave deep clarity on:

PKI structure

Mutual TLS handshakes

TLS passthrough vs TLS termination

Real enterprise API security patterns 


## 🔐 1. Why NLB Instead of ALB?
```bash
Client ──TLS──▶ NLB (TCP passthrough) ──TLS──▶ EC2/nginx
     ▲                                         │
     │                                         │ validates client cert
     │                                         ▼
 Client certificate                     Server certificate
 signed by                              signed by
 Intermediate CA                         Intermediate CA
            ▲                                     ▲
            │                                     │
          Root CA ——— signs ——— Intermediate CA ——— signs ——— server/client certs
```
🟢 TLS session is end-to-end (client ↔ server)

🟢 NLB blindly forwards encrypted bytes (no decryption)

🟢 NLB does NOT need ACM

🟢 Server validates client certificate

🟢 Most secure possible banking model

| Feature                     | ALB (Layer 7)     | NLB (Layer 4)                    |
| --------------------------- | ----------------- | -------------------------------- |
| TLS termination             | ✔ Yes (always)    | ❌ No (unless using TLS listener) |
| Requires ACM cert           | ✔ Yes             | ❌ No (TCP passthrough)           |
| mTLS supported              | ❌ No              | ✔ Yes                            |
| Pass client cert to backend | ❌ No              | ✔ Yes                            |
| Handles non-HTTP TLS        | ❌ No              | ✔ Yes                            |
| Client IP preserved         | ❌ Uses XFF header | ✔ Native L4 source IP            |



#### `ALB breaks mTLS` because it terminates TLS and the backend never sees the client certificate.

#### `NLB preserves mTLS` because it forwards encrypted bytes directly.

## 2. PKI Architecture (Root CA → Intermediate CA)

Banks NEVER sign server/client certificates directly from root CA.

Instead they use:
```bash
Root CA (offline)
   ↓
Intermediate CA (online and signed by Root CA)
   ↓
Server certs  
Client certs
```

This way, the root CA stays secure.

We will implement the same structure.


## 3. Step 1 – Create Directory Structure

```bash
mkdir -p ~/mtls-lab/{rootCA,intermediateCA,server,client}
cd ~/mtls-lab
```

## 4. Step 2 – Create Root CA
#### Generate root key
```bash
openssl genrsa -out rootCA/rootCA.key 4096
```

#### Create self-signed root certificate
```bash
openssl req -x509 -new -nodes -key rootCA/rootCA.key \
    -sha256 -days 3650 \
    -out rootCA/rootCA.crt \
    -subj "/C=IN/ST=KA/L=Bangalore/O=BankRootCA/OU=Root/CN=BankRootCA"
```

## 5. Step 3 – Create Intermediate CA
#### Intermediate Key
```bash
openssl genrsa -out intermediateCA/intermediate.key 4096
```

#### intermediate csr
```bash
openssl req -new -key intermediateCA/intermediate.key \
    -out intermediateCA/intermediate.csr \
    -subj "/C=IN/ST=KA/L=Bangalore/O=BankIntermediateCA/OU=CA/CN=IntermediateCA"
```

#### Sign intermediate CA using root CA
```bash
openssl x509 -req \
    -in intermediateCA/intermediate.csr \
    -CA rootCA/rootCA.crt \
    -CAkey rootCA/rootCA.key \
    -CAcreateserial \
    -out intermediateCA/intermediate.crt \
    -days 3650 -sha256 \
    -extfile <(printf "basicConstraints=CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign")
```

## 6. Step 4 – Build Full CA Chain
```bash
cat intermediateCA/intermediate.crt rootCA/rootCA.crt > intermediateCA/ca-chain.crt
```
Used by:

* nginx to verify the client certificate

* curl to verify server certificate

## 7. Step 5 – Create Server Certificate (for nginx)
#### server key
```bash
openssl genrsa -out server/server.key 2048
```

#### server csr
```bash
openssl req -new -key server/server.key \
    -out server/server.csr \
    -subj "/C=IN/ST=KA/L=Bangalore/O=MyBankAPI/OU=Server/CN=server.example.com"
```

#### signed server crt
```bash
openssl x509 -req \
    -in server/server.csr \
    -CA intermediateCA/intermediate.crt \
    -CAkey intermediateCA/intermediate.key \
    -CAcreateserial \
    -out server/server.crt \
    -days 825 -sha256 \
    -extfile <(printf "extendedKeyUsage=serverAuth")
```

## 8. Step 6 – Create Client Certificate
#### client key
```bash
openssl genrsa -out client/client.key 2048
```

#### client csr
```bash
openssl req -new -key client/client.key \
    -out client/client.csr \
    -subj "/C=IN/ST=KA/L=Bangalore/O=MyBankClient/OU=Client/CN=client1"
```

#### signed client crt
```bash
openssl x509 -req \
    -in client/client.csr \
    -CA intermediateCA/intermediate.crt \
    -CAkey intermediateCA/intermediate.key \
    -CAcreateserial \
    -out client/client.crt \
    -days 825 -sha256 \
    -extfile <(printf "extendedKeyUsage=clientAuth")
```

## 9. Step 7. Copy the server's key and crt into nginx server 
```bash
$ cp -i /home/anuroop/Downloads/alb-kp.pem /home/anuroop/mtls-lab/server/server.key ec2-user@98.81.29.70:/home/ec2-user/

$ cp -i /home/anuroop/Downloads/alb-kp.pem /home/anuroop/mtls-lab/server/server.crt ec2-user@98.81.29.70:/home/ec2-user/

$ cp -i /home/anuroop/Downloads/alb-kp.pem /home/anuroop/mtls-lab/intermediateCA/ca-chain.crt   ec2-user@98.81.29.70:/home/ec2-user/
```

### Inside nginx server
```bash

$ sudo mkdir -p /etc/nginx/ssl

$ sudo mv server.crt /etc/nginx/ssl/

$ sudo mv server.key /etc/nginx/ssl/

$ sudo mv ca-chain.crt /etc/nginx/ssl/

$ sudo chmod 600 /etc/nginx/ssl/ca-chain.crt
```

## 10. Step 8 – Configure nginx for mTLS
#### nginx.conf

Inside http {} block:
```bash
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    ssl_client_certificate /etc/nginx/ssl/ca-chain.crt;
    ssl_verify_client on;

    ssl_verify_depth 2;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        return 200 "SUCCESS: mTLS handshake completed on EC2. Client certificate validated.\n";
    }
}
```

## 11. Step 9 – Test mTLS Locally (Before NLB)

#### ❌ WITHOUT client certificate:
```bash
curl -vk https://<EC2-IP>
```

Expected:
```bash
400 No required SSL certificate was sent
```

#### ✔ WITH client certificate:
```bash
curl -vk \
  --cert client/client.crt \
  --key client/client.key \
  https://<EC2-IP>
```

Expected:
```bash
SUCCESS: mTLS handshake completed on EC2.
```

## 12. Step 10 – Create NLB (TCP 443 Listener)
#### Listener:
```bash
TCP : 443
```

#### Target Group:
```bash
Protocol: TCP
Port: 443
Health Check: TCP
```

Register your instances.

## 13. Step 11 – Test mTLS via NLB
Let NLB DNS be:
```bash
my-nlb-12345.elb.amazonaws.com
```
#### ❌ Test without client cert:
```bash
curl -vk https://<NLB-DNS>
```

Should fail:
```bash
400 No required SSL certificate was sent
```
#### ✔ Test with client cert:
```bash
curl -vk \
  --cert client/client.crt \
  --key client/client.key \
  https://<NLB-DNS>
```

Success response:
```bash
SUCCESS: mTLS handshake completed on EC2.
```

#### 🎉 END-TO-END mTLS over NLB works!

## 14. Real-World Use Cases
| Use Case               | Why NLB mTLS                  |
| ---------------------- | ----------------------------- |
| Banking APIs           | Client certificates mandatory |
| UPI Switch             | mTLS between PSP & NPCI       |
| Secure B2B APIs        | Verify partner identity       |
| Government systems     | Aadhaar, GST, NIC APIs        |
| Internal microservices | Zero trust authentication     |
| VPN / Tunnels          | TLS passthrough required      |



## Appendix
```bash
mtls-lab/
├── rootCA/
│   ├── rootCA.key
│   ├── rootCA.crt
├── intermediateCA/
│   ├── intermediate.key
│   ├── intermediate.crt
│   ├── ca-chain.crt
├── server/
│   ├── server.key
│   ├── server.csr
│   ├── server.crt
├── client/
│   ├── client.key
│   ├── client.csr
│   ├── client.crt
```


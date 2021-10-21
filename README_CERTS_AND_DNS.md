## Ansible Deployment 

We will be using Ansible playbooks to install and configure the ELK stack components.
Ansible is available in the package repositories fpr Liux and Mac.

### SSL Cert with acme.sh
We already set up our DNS server to point to the deployd EC2s. We will also need a valid SSL certificates for our ELK stack deployment.
In this example here we're using the Acme.sh API utility and the GoDaddy provider.
[acme.sh](https://github.com/acmesh-official/acme.sh)
[Use GoDaddy.com domain API to automatically issue cert](https://github.com/acmesh-official/acme.sh/wiki/dnsapi#4-use-godaddycom-domain-api-to-automatically-issue-cert)

```shell
export GD_Key="xxx"
export GD_Secret="PPPX123"
```
```shell
acme.sh --issue --dns dns_gd -d kibana.sciviz.co -d es01.sciviz.co -d es02.sciviz.co -d es03.sciviz.co

[Tue Feb  2 07:42:03 PM CST 2021] Your cert is in  /home/user/.acme.sh/kibana.sciviz.co/kibana.sciviz.co.cer 
[Tue Feb  2 07:42:03 PM CST 2021] Your cert key is in  /home/user/.acme.sh/kibana.sciviz.co/kibana.sciviz.co.key 
[Tue Feb  2 07:42:03 PM CST 2021] The intermediate CA cert is in  /home/user/.acme.sh/kibana.sciviz.co/ca.cer 
[Tue Feb  2 07:42:03 PM CST 2021] And the full chain certs is there:  /home/user/.acme.sh/kibana.sciviz.co/fullchain.cer 
 
```
Similarly if your DNS is managed by AWS 
```shell

export  AWS_ACCESS_KEY_ID=AKxxxx
export  AWS_SECRET_ACCESS_KEY=gKyyyy
```
```shell
acme.sh --issue --dns dns_aws -d kibana.sciviz.co -d es01.sciviz.co -d es02.sciviz.co -d es03.sciviz.co
```
[letsencrypt + route53](https://gist.github.com/nelsonenzo/35a95107cee1a57e7ac4178c526b1b00)
### Generate the PKCS12 (p12, aka pfx) file 
Go to the folder where the certificates were generated with `acme.sh`
```shell
openssl pkcs12 -export  -inkey kibana.sciviz.co.key  -in kibana.sciviz.co.cer -certfile fullchain.cer -out kibana.sciviz.co.p12 
```

```shell
export  SSL_DOMAIN=kibana.sciviz.co
export SSL_PASS=changeme
openssl pkcs12 -export \
-inkey ${SSL_DOMAIN}.key \
-in ${SSL_DOMAIN}.cer \
 -certfile fullchain.cer  \
-out ${SSL_DOMAIN}.pfx   \
-password env:SSL_PASS
 
```
[https://www.openssl.org/docs/man1.1.0/man1/openssl.html#Pass-Phrase-Options](https://www.openssl.org/docs/man1.1.0/man1/openssl.html#Pass-Phrase-Options)
            

### GoDaddy DNS Change

```shell
export GD_Key="xxx"
export GD_Secret="PPPX123"
``` 

##### DNS Update GoDaddy Nameservers
We set up with Terraform a Private and Public hosted zones. We would need to point our GoDaddy domain to the Custom Nameservers. We take these values
AWS Console>Route53>Public Zone>NS Type  Value/Route traffic to
These nameserver have the following format
```shell
ns-234.awsdns-33.com.
ns-999.awsdns-59.net.
ns-1234.awsdns-55.co.uk.
ns-123.awsdns-22.org.
```
We use the GoDaddy API for the custom DNS Nameserver update.
```shell

curl -X PATCH https://api.godaddy.com/api/v1/domains/sciviz.co \
-H "Authorization: sso-key ${GD_Key}:${GD_Secret}" \
-H "Content-Type: application/json" \
--data  '{	"nameServers": [
"ns-1143.awsdns-14.org",
"ns-154.awsdns-19.com",
"ns-1828.awsdns-36.co.uk",
"ns-576.awsdns-08.net"
]
  }'
  ```
##### Check if the change was propagated
```shell
curl -X GET  https://api.godaddy.com/api/v1/domains/sciviz.co \
-H "Authorization: sso-key ${GD_Key}:${GD_Secret}" \
  -H "Content-Type: application/json" 
```

##### DNS Set A record

For documentation purposes I am posting also how to change the DNS to the hostnames directly.
```shell
curl -X PATCH https://api.godaddy.com/v1/domains/sciviz.co/records \
-H "Authorization: sso-key ${GD_Key}:${GD_Secret}" \
  -H "Content-Type: application/json" \
  --data '[
  {"type": "A","name": "es1","data": "34.209.128.5","ttl": 3600},
   {"type": "A","name": "es2","data": "52.38.139.226","ttl": 3600},
   {"type": "A","name": "es3-sb","data": "34.209.124.156","ttl": 3600}
  ]'

```
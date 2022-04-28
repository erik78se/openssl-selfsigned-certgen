# openssl-selfsigned-certgen
Create openssl self signed certs

## Run
    ./create-ssl-cert.sh -cn test.example.com

## Overview
The repo has a script that generates a single openssl certificate and corresponding .key and a fullchain.pem certificate chain. (The fullchain.pem is basically the concatenation of the cert + key)

The certificates are placed in a certs folder like this:

    certs/
    ├── fullchain.pem
    ├── server.crt
    ├── server.csr
    └── server.key

## Use with juju haproxy charm
If you like to try it out, use this charm https://charmhub.io/microsample

Together with https://charmhub.io/haproxy

### Deploy the example service and relate it with the haproxy

    juju deploy microsample
    juju add-unit microsample
    juju deploy haproxy
    juju relate microsample:website haproxy:reverseproxy

### Open up ports for the haproxy (if needed)
    juju expose haproxy

### Inject the certificates you created
    juju config haproxy ssl_cert="$(base64 server.pem)"
    juju config haproxy ssl_key="$(base64 server.key)"

### Compose a haproxy [services.yaml](services.yaml)
While this file might seem daunting, its really just config stanzas for /etc/haproxy/haproxy.cfg and you can probably create your own with some trial and error with juju config.
<pre>
- service_name: myservice
  service_host: "0.0.0.0"
  crts: [DEFAULT]
  service_port: 443
  service_options: 
    - http-request set-header X-Forwarded-Port %[dst_port]
    - http-request add-header X-Forwarded-Proto https if { ssl_fc }
    - balance leastconn
    - option forwardfor
    - cookie SRVNAME insert
    - acl url_discovery path /.well-known/caldav /.well-known/carddav
    - http-request redirect location /remote.php/dav/ code 301 if url_discovery
  server_options: 
    - maxconn 100 cookie S{i} check
</pre>

### Supply the [services.yaml](services.yaml) to haproxy
    juju config haproxy services="$(cat services.yaml)"

### Verify that the configuration has worked.
If everything has worked, *juju status* will show port 443 opened instead of 80 and haproxy is active.

'haproxy/0*  active    idle   1        192.168.111.233  443/tcp   Unit is ready'

### Test
You can now test that the service is responding with curl:

    curl -k https://192.168.111.233
    Online

Congratulations!

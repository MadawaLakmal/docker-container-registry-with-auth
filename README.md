Guide: "http://jgsqware.github.io/2016/02/docker-registry-installation/"

OpenSSL: "https://devopscube.com/create-self-signed-certificates-openssl/"

### Generate CA
```
openssl req -x509 \
            -sha256 -days 3560 \
            -nodes \
            -newkey rsa:2048 \
            -subj "/CN=*.madawa.com/C=SL/L=COLOMBO" \
            -keyout rootCA.key -out rootCA.crt
            
```
            
### Generate cert and key

```
1. openssl genrsa -out server.key 2048

2.
cat > csr.conf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
C = SL
ST = Western
L = Colombo
O = Madawa Inc.
OU = DigiOps
CN = registry.madawa.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = registry.madawa.com
DNS.2 = www.registry.madawa.com
IP.1 = 10.168.8.200

EOF

3. openssl req -new -key server.key -out server.csr -config csr.conf

4.
cat > cert.conf <<EOF

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = registry.madawa.com
IP.1 = 10.168.8.200

EOF

5.
openssl x509 -req \
    -in server.csr \
    -CA rootCA.crt -CAkey rootCA.key \
    -CAcreateserial -out server.crt \
    -days 3650 \
    -sha256 -extfile cert.conf
    
    
# If we are using self-sign certificates make sure to use your own CA, because there is a bug in a OpenSSL 1.1.1 version with GO. Also add the rootCA into your localhost(/usr/local/share/ca-certificates/rootCA.crt) and /etc/docker/docker.registry:5000/ directory as ca.crt.  
# Also follow the same steps to generate auth.madawa.com cert and key

```

### sample-config.yml,

```
server:  # Server settings.
  # Address to listen on.
  addr: ":5001"
  # TLS certificate and key.
  certificate: "/ssl/auth-server.crt"
  key: "/ssl/auth-server.key"

token:  # Settings for the tokens.
  issuer: "auth_service"  # Must match issuer in the Registry config.
  expiration: 900


# Static user map.
users:
  # Password is specified as a BCrypt hash. Use htpasswd -B to generate.
  "admin":
    password: "$2y$05$egkZSRgEpr0MEHfuhcI7KehPUveatklaBOVuaDnEKMaZKSBEM66Qi"
  "madawa": # my user
    password: "$2y$05$bD1sxGVeEt5FuxXJn.NPp.E2.xWEqm.6qM7ASCQUSqAQelhXhxTHe"

acl:
  # Admin has full access to everything.
  - match: {account: "admin"}
    actions: ["*"]
  # Users have full right on their repository
  - match: {account: "/.+/", name: "${account}/*"}
    actions: ["*"]
  - match: {account: "madawa"}
    actions: ["pull"]
  # Access is denied by default.
```

### sample docker-compose.yml,

```
version: '2'

services:
  auth:
    image: cesanta/docker_auth
    ports:
      - "5001:5001"
    volumes:
      - ./config:/config:ro
      - ./ssl:/ssl
    command: /config/auth_config.yml
    container_name: "auth"

  registry:
    image: registry:latest
    ports:
      - "5000:5000"
    volumes:
      - ./ssl:/ssl
      - ./registry-ssl:/registry-ssl
      - registry-data:/var/lib/registry
    container_name: "registry"
    environment:
      - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry
      - REGISTRY_AUTH=token
      - REGISTRY_AUTH_TOKEN_REALM=https://auth.madawa.com:5001/auth # the authentication server URI
      - REGISTRY_AUTH_TOKEN_SERVICE="registry"
      - REGISTRY_AUTH_TOKEN_ISSUER="auth_service" # Should be the same as token.issuer from authenticated-registry/config/auth_config.yml
      - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/ssl/auth-server.crt
      - REGISTRY_HTTP_TLS_CERTIFICATE=/registry-ssl/wild/registry-server.crt
      - REGISTRY_HTTP_TLS_KEY=/registry-ssl/wild/registry-server.key
      - REGISTRY_STORAGE_DELETE_ENABLED=true

volumes:
  registry-data: # the real name will be <parent-folder-name>_registry-data (dash '-' int he folder name will be removed). eg. authenticatedregistry_registry-data
    driver: local
```

Make sure to copy rootCA.crt to your local machines /usr/local/share/ca-certificates/ca.crt path and as well as create a directory inside your docker client machine
following path as naming the registry /etc/docker/registry.madawa.com:5000 and copy the rootCA.crt inside that directory.

# responding to acme challenges with haproxy

## background

I deploy [`haproxy`](https://github.com/haproxy/haproxy) in front of [`apache2`](https://github.com/apache/httpd) because I like to offer ssh-over-https (using a custom ALPN value for routing) as a means of remoting into my servers in order to avoid silly outbound firewall rules that block port 22.

This approach also gives me the flexibility of deploying different web servers (apache2, nginx, etc.) to handle different web sites (as part of my learning journey).

Of course, I desire that these web sites be available via HTTPS and I leverage [Let's Encrypt](https://letsencrypt.org/) for certificates, which I obtain using [`dehydrated`](https://github.com/dehydrated-io/dehydrated). I terminate the TLS connections at haproxy.

In order to respond to the ACME challenges, I have been using [apache2-dehydrated](https://packages.debian.org/bullseye/dehydrated-apache2), Debian's packaging of dehydrated's [example apache2 configuration](https://github.com/dehydrated-io/dehydrated/blob/5c1551e946456f534cf46b6ebabe4353bf0b0530/docs/wellknown.md#apache-example-config) which supports responding to ACME challenges without configuring an apache2 virtual host.

But it has always bothered me that I needed apache2 to respond to ACME challenges for certificates used by haproxy... so I did something about it.

The key points:

* haproxy can be configured to [expose an admin socket](http://docs.haproxy.org/2.6/configuration.html#stats%20socket) over which commands may be issued, including the [addition](http://docs.haproxy.org/2.6/management.html#add%20map) and [deletion](http://docs.haproxy.org/2.6/management.html#del%20map) of entries into an existing map
* dehydrated can be configured to invoke a hook script, which it will call at various points during the process of obtaining a certificate; two of these hook points (deploy_challenge and clean_challenge) can be used to issue commands to haproxy via it's admin socket to update the map with responses to ACME challenges
* haproxy's [`http-request return`](http://docs.haproxy.org/2.6/configuration.html#http-request%20return) directive can be used to respond to client requests without forwarding to a backend and whose `lf-string` parameter can look up responses in the hook-maintained ACME map when the request's path begins with `/.well-known/acme-challenge/`
* haproxy's [`ssl-load-extra-files`](http://docs.haproxy.org/2.6/configuration.html#3.1-ssl-load-extra-files) directive can be used to instruct haproxy to look for cert bundles named as follows:
    * «basename».\[rsa|ecdsa\] - contains the end-entity certificate and intermediate certificates (if any); equivalent to dehydrated's fullchain.pem
    * «basename».\[rsa|ecdsa\].key - contains the end-entity private key; equivalent to dehydrated's privkey.pem
    * «basename».\[rsa|ecdsa\].ocsp - contains the OCSP; equivalent to dehydrated's ocsp.der

Additional observations:
* dehydrated allows [multiple aliases per domain](https://github.com/dehydrated-io/dehydrated/blob/master/docs/domains_txt.md#aliases)
* dehydrated allows [alternate configurations per alias](https://github.com/dehydrated-io/dehydrated/blob/master/docs/per-certificate-config.md)
* the above two dehydrated featues are used to obtain both RSA and ECDSA certificates for a domain

## configuration

### dehydrated

#### /etc/dehydrated/config

```toml
BASEDIR="/var/lib/dehydrated"
CONFIG_D="/etc/dehydrated/conf.d"
DOMAINS_D="/etc/dehydrated/domains.d"
DOMAINS_TXT="/etc/dehydrated/domains.txt"
HOOK="/etc/dehydrated/hook.sh"
OCSP_FETCH="yes"
OCSP_MUST_STAPLE="yes"
RENEW_DAYS=35
WELLKNOWN="${BASEDIR}/acme-challenges"
```

#### /etc/dehydrated/domains.txt

Specify [multiple aliases per domain](https://github.com/dehydrated-io/dehydrated/blob/master/docs/domains_txt.md#aliases) to leverage dehydrated's [alternate configurations per alias](https://github.com/dehydrated-io/dehydrated/blob/master/docs/per-certificate-config.md) feature to obtain both RSA and ECDSA certificates for a domain.

```text
# the format is:
#   domainPrimaryName [, domainAlternameName(s)] > alias
# multiple aliases per domainPrimaryName are permitted
example.com > example.com.ecc
example.com > example.com.rsa
example.org > example.org.ecc
example.org > example.org.rsa
```

#### /etc/dehydrated/domains.d/*

Specify RSA or ECDSA key generation using dehydrated's [alternate configurations per alias](https://github.com/dehydrated-io/dehydrated/blob/master/docs/per-certificate-config.md) feature.

| filename        | contents               |
| --------------- | ---------------------- |
| example.com.ecc | `KEY_ALGO="secp384r1"` |
| example.com.rsa | `KEY_ALGO="rsa"`       |
| example.org.ecc | `KEY_ALGO="secp384r1"` |
| example.org.rsa | `KEY_ALGO="rsa"`       |

#### /etc/dehydrated/hook.sh

```bash
#!/bin/bash
# Copyright (c) 2023 Luca Filipozzi

declare -A alg2ext=( ["rsaEncryption"]="rsa" ["id-ecPublicKey"]="ecdsa" )

deploy_challenge() {
    local DOMAIN="${1}" TOKEN="${2}" VALUE="${3}"
    echo "add map /etc/haproxy/acme.map ${TOKEN} ${VALUE}" | socat stdio unix-connect:/run/haproxy/admin.sock
}

clean_challenge() {
    local DOMAIN="${1}" TOKEN="${2}" VALUE="${3}"
    echo "del map /etc/haproxy/acme.map ${TOKEN}" | socat stdio unix-connect:/run/haproxy/admin.sock
}

deploy_cert() {
    local DOMAIN="${1}" PRIVKEY="${2}" CERT="${3}" FULLCHAIN="${4}" CHAIN="${5}" TIMESTAMP="${6}"
    local SRC=$(dirname ${PRIVKEY})
    local DST=/var/lib/haproxy/certs
    local ALG=$(openssl x509 -in ${SRC}/cert.pem -noout -text | awk -F':' '/Public Key Algorithm/ {print $2}' | tr -d ' ')
    local EXT=${alg2ext[${ALG}]}
    ln -sf ${FULLCHAIN} ${DST}/${DOMAIN}.${EXT}
    ln -sf ${PRIVKEY}   ${DST}/${DOMAIN}.${EXT}.key
}

deploy_ocsp() {
    local DOMAIN="${1}" OCSP="${2}" TIMESTAMP="${3}"
    local SRC=$(dirname ${OCSP})
    local DST=/var/lib/haproxy/certs
    local ALG=$(openssl x509 -in ${SRC}/cert.pem -noout -text | awk -F':' '/Public Key Algorithm/ {print $2}' | tr -d ' ')
    local EXT=${alg2ext[${ALG}]}
    ln -sf ${OCSP}      ${DST}/${DOMAIN}.${EXT}.ocsp
}

unchanged_cert() {
    :
}

invalid_challenge() {
    :
}

request_failure() {
    :
}

generate_csr() {
    :
}

startup_hook() {
    mkdir -p /var/lib/haproxy/certs
}

exit_hook() {
    systemctl reload-or-restart haproxy
}

HANDLER="$1"; shift
if [[ "${HANDLER}" =~ ^(deploy_challenge|clean_challenge|deploy_cert|deploy_ocsp|unchanged_cert|invalid_challenge|request_failure|generate_csr|startup_hook|exit_hook)$ ]]; then
    "$HANDLER" "$@"
fi
```

### haproxy

#### /etc/haproxy/acme.map

```plaintext
# intentionally empty
```

#### /etc/haproxy/cert.lst

Only need to specify the basename when using haproxy's [ssl-load-extra-files](http://docs.haproxy.org/2.6/configuration.html#ssl-load-extra-files) feature.

```plaintext
/var/lib/haproxy/certs/example.com
/var/lib/haproxy/certs/example.org
```

#### /etc/haproxy/haproxy.cfg

```haproxy
global
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners

frontend http-ingress
    bind :80
    acl is_acme path_dir /.well-known/acme-challenge/
    http-request redirect scheme https code 301 unless is_acme
    http-request return status 200 content-type application/octet-stream lf-string %[path,map_end(/etc/haproxy/acme.map,nope)] if is_acme
    http-request deny deny_status 500
    
frontend https-ingress
    bind :443 strict-sni ssl crt-list /etc/haproxy/cert.lst ecdhe secp384r1
    default_backend apache2
```

---
Copyright (c) 2023 Luca Filipozzi

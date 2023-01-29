# responding to acme challenges with haproxy

## background

I deploy [`haproxy`](https://github.com/haproxy/haproxy) in front of [`apache2`](https://github.com/apache/httpd) because I like to offer ssh-over-https (using a custom ALPN value for routing) as a means of remoting into my servers in order to avoid silly outbound firewall rules that block port 22.

This approach also gives me the flexibility of deploying different web servers (apache2, nginx, etc.) to handle different web sites (as part of my learning journey).

Of course, I desire that these web sites be available via HTTPS and I leverage [Let's Encrypt](https://letsencrypt.org/) for certificates, which I obtain using [`dehydrated`](https://github.com/dehydrated-io/dehydrated). I terminate the TLS connections at haproxy.

In order to respond to the ACME challenges, I have been using [apache2-dehydrated](https://packages.debian.org/bullseye/dehydrated-apache2), Debian's packaging of dehydrated's [example apache2 configuration](https://github.com/dehydrated-io/dehydrated/blob/5c1551e946456f534cf46b6ebabe4353bf0b0530/docs/wellknown.md#apache-example-config) which supports responding to ACME challenges without configuring an apache2 virtual host.

But it has always bothered me that I needed apache2 to respond to ACME challenges for certificates used by haproxy... so I did something about it.

The key points:

* haproxy can be configured to expose an administrative socket over which commands may be issued, including the [addition](https://www.haproxy.com/documentation/hapee/latest/api/runtime-api/add-map/) and [deletion](https://www.haproxy.com/documentation/hapee/latest/api/runtime-api/del-map/) of entries into an existing map
* dehydrated can be configured to invoke a hook script, which it will call at various points during the process of obtaining a certificate; two of these hook points (deploy_challenge and clean_challenge) can be used to issue commands to haproxy via it's administrative coscket
* haproxy's `http-request respond` directive can be used to respond to client requests without forwarding to a backend, which can be look up responses in the hook-maintained map

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

```text
example.com > example.com.ecdsa
example.com > example.com.rsa
example.org > example.org.ecdsa
example.org > example.org.rsa
```

#### /etc/dehydrated/domains.d/*

| filename          | contents               |
| ----------------- | ---------------------- |
| example.com.ecdsa | `KEY_ALGO="secp384r1"` |
| example.com.rsa   | `KEY_ALGO="rsa"`       |
| example.org.ecdsa | `KEY_ALGO="secp384r1"` |
| example.org.rsa   | `KEY_ALGO="rsa"`       |

#### /etc/dehydrated/hook.sh

```bash
#!/bin/bash

deploy_challenge() {
    local DOMAIN="${1}" TOKEN="${2}" VALUE="${3}"
    echo "add map  /etc/haproxy/acme.map ${TOKEN} ${VALUE}" | socat stdio unix-connect:/run/haproxy/admin.sock
}

clean_challenge() {
    local DOMAIN="${1}" TOKEN="${2}" VALUE="${3}"
    echo "del map /etc/haproxy/acme.map ${TOKEN}" | socat stdio unix-connect:/run/haproxy/admin.sock
}

deploy_cert() {
    local DOMAIN="${1}" KEY="${2}" CRT="${3}" CHN="${5}" ALIAS="$(basename $(dirname ${2}))"
    local OUT="$(dirname ${2})/haproxy.pem.${ALIAS##*.}"
    cat ${CRT} ${CHN} ${KEY} > ${OUT}
}

deploy_ocsp() {
    local DOMAIN="${1}" DER="${2}" TS="${3}" ALIAS="$(basename $(dirname ${2}))"
    local OUT="$(dirname ${2})/haproxy.pem.${ALIAS##*.}.ocsp"
    cat ${DER} > ${OUT}
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
    :
}

exit_hook() {
    systemctl reload-or-restart haproxy
    :
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

#### /etc/haproxy/haproxy.cfg

```haproxy
global
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners

frontend ingress
    bind :80
    acl is_acme path_dir /.well-known/acme-challenge/
    http-request redirect scheme https code 301 unless is_acme
    http-request return status 200 content-type application/octet-stream lf-string %[path,map_end(/etc/haproxy/acme.map,nope)] if is_acme
    http-request deny deny_status 500
```

---
Copyright (c) 2023 Luca Filipozzi

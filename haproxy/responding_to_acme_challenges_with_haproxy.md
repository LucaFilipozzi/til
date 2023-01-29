# responding to acme challenges with haproxy

I deploy haproxy in front of apache because I like to offer ssh-over-https (using a custom ALPN value) as a means of remoting into my servers in order to avoid silly outbound port-blocking rules.

This approach also gives me the flexibility of deploying different web servers (apache2, nginx, etc.) to handle different websites (part of my learning journey).

Of course, I desire that these websites be available via HTTPS and I leverage Let's Encrypt for certificates, which I obtain using [`dehydrated`](https://github.com/dehydrated-io/dehydrated).

In order to respond to the ACME challenge, I have been using [apache2-dehydrated](https://packages.debian.org/bullseye/dehydrated-apache2), Debian's packaging of dehydrated's [example apache2 configuration](https://github.com/dehydrated-io/dehydrated/blob/5c1551e946456f534cf46b6ebabe4353bf0b0530/docs/wellknown.md#apache-example-config) to support responding to challenges without configuring an virtual host.

But it has always bothered me that I needed apache2 to respond to ACME challenges for certificates used by haproxy... so I did something about it.

# dehydrated hook script

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
    local OUT="$(dirname ${2})/haproxy.pem.${ALIAS##*.}" LST="/etc/haproxy/cert.lst"
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

---
Copyright (c) 2023 Luca Filipozzi

# Let's Encrypt

### Prepare credentials

Before we can do anything, generate an account key first if you don't have one.

    $ openssl genrsa -out account.key 4096

Generate a domain key and CSR.

    $ openssl genrsa -out domain.key 4096
    $ openssl req -key domain.key -new -sha256 -subj "/CN=example.com" -out domain.csr

Alternatively, if you need a SAN certificate.

    $ openssl req -key domain.key -new -sha256 -subj "/" -out domain.csr \
        -reqexts SAN \
        -config <(cat /path/to/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:example.com,DNS:www.example.com"))

You can use the same CSR across multiple renewals. But you can't use your
account private key as your domain private key!

### Prepare your web server

Let's Encrypt now requires hosting some files on the domains you requested to
prove that you control the domain. Those files should be served at
`/.well-known/acme-challenge/` URL path, on port 80.

Create a dedicate directory.

    $ mkdir -p /srv/letsencrypt/challenges/

Serve challenges from the above directory.

    server {
        listen 80;
        server_name example.com, www.example.com;

        location /.well-known/acme-challenge/ {
            alias /srv/letsencrypt/challenges/;
            try_files $uri =404;
        }
    }

### Get a certificate

Run `letsencrypt` on your server with proper permissions to write to the above
folder and read your private account key and CSR.

    $ ./letsencrypt --account-key account.key --email admin@example.com --challenge-dir /srv/letsencrypt/challenges/ domain.csr > domain.crt

### Install the certificate

Bundle `domain.crt` with the **Let’s Encrypt Authority X3** intermediate certificate.

    $ curl https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem -o intermediate.crt
    $ cat domain.crt intermediate.crt > chained.crt

Then enable SSL.

    server {
        listen 443 ssl http2;
        server_name example.com, www.example.com;

        ssl_certificate /path/to/chained.crt;
        ssl_certificate_key /path/to/domain.key;
    }

    server {
        listen 80;
        server_name yoursite.com, www.yoursite.com;

        location /.well-known/acme-challenge/ {
            alias /srv/letsencrypt/challenges/;
            try_files $uri =404;
        }
    }

### Setup an auto-renew crontab job

Let's Encrypt certificates only last for 90 days. It is recommended to renew
certificates every 60 days.

Create a `renew.sh`.

    #!/usr/bin/sh

    # request certificate
    /path/to/letsencrypt --account-key /path/to/account.key --email admin@example.com --challenge-dir /srv/letsencrypt/challenges/ /path/to/domain.csr > /path/to/domain.crt
    # make chain
    curl https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem -o /path/to/intermediate.crt
    cat /path/to/domain.crt /path/to/intermediate.crt > /path/to/chained.crt
    # reload nginx
    sudo service nginx reload

Then add it to crontab.

    0 0 1 */2 * /path/to/renew_cert.sh

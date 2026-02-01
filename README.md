# root-certificate-authority

1) Make a place for certs
```bash
sudo mkdir -p /etc/nginx/ssl
sudo chmod 700 /etc/nginx/ssl
cd /etc/nginx/ssl
```

2) Create a Root CA (10-year)
```bash
# Root CA private key (keep this SECRET)
openssl genrsa -out rootCA.key 4096

# Self-signed Root CA certificate (PEM)
openssl req -x509 -new -nodes -sha256 -days 3650 \
  -key rootCA.key \
  -subj "/C=NP/ST=Bagmati/L=Kathmandu/O=HomeLab CA/OU=Root/CN=mele.box Root CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign" \
  -addext "subjectKeyIdentifier=hash" \
  -out rootCA.pem

# You can also keep a .crt copy if you like
cp rootCA.pem rootCA.crt
```

3) Create the server key + CSR for mele.box
Create an extensions file to ensure SAN and proper key usage:
```bash
cat > mele.box.ext <<'EOF'
authorityKeyIdentifier = keyid,issuer
basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, keyEncipherment
extendedKeyUsage       = serverAuth
subjectAltName         = @alt_names

[alt_names]
DNS.1 = mele.box
DNS.2 = *.mele.box
EOF
```
Now generate key + CSR:
```bash
# Server private key
openssl genrsa -out mele.box.key 2048

# CSR with CN=mele.box (SAN comes from the .ext file, not here)
openssl req -new -sha256 \
  -key mele.box.key \
  -subj "/CN=mele.box" \
  -out mele.box.csr
```

4) Issue the server certificate from your Root CA
```bash
openssl x509 -req -sha256 -days 825 \
  -in mele.box.csr \
  -CA rootCA.pem -CAkey rootCA.key \
  -CAcreateserial \
  -extfile mele.box.ext \
  -out mele.box.crt
```

5) Configure Nginx (minimal change)
```bash
# /etc/nginx/sites-available/mele.box (or your existing server file)

server {
    listen 80;
    server_name mele.box;
    return 301 https://$host$request_uri;  # optional redirect to HTTPS
}

server {
    listen 443 ssl http2;
    server_name mele.box;

    ssl_certificate     /etc/nginx/ssl/mele.box.crt;
    ssl_certificate_key /etc/nginx/ssl/mele.box.key;

    # (Optional but good defaults)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://localhost:8089;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

6) Trust your Root CA on clients
```bash
# Use /usr/local/share/ca-certificates with a .crt extension
sudo cp /etc/nginx/ssl/rootCA.pem /usr/local/share/ca-certificates/mele-rootCA.crt
sudo update-ca-certificates
```

7) Verify end-to-end
```bash
curl -I https://mele.box
```

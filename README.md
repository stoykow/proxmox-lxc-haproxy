# Proxmox LXC HAProxy Setup

## HAProxy Beispiel-Konfiguration

### HTTP Frontend
```
frontend http_in
    bind *:80
    mode http

    acl letsencrypt_http path_beg /.well-known/acme-challenge/
    http-request redirect scheme https code 301 if !letsencrypt_http
    use_backend letsencrypt_backend if letsencrypt_http
```

### HTTPS Frontend
```
frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/
    mode http

    acl host_help hdr(host) -i help.domain.ltd
    acl host_qiskit hdr(host) -i qiskit.domain.ltd
    acl host_z2m1 hdr(host) -i zigbee2mqtt-1.domain.ltd
    acl host_z2m2 hdr(host) -i zigbee2mqtt-2.domain.ltd
    acl host_z2m3 hdr(host) -i zigbee2mqtt-3.domain.ltd

    use_backend glpi_backend if host_help
    use_backend qiskit_backend if host_qiskit
    use_backend zigbee2mqtt-1_backend if host_z2m1
    use_backend zigbee2mqtt-2_backend if host_z2m2
    use_backend zigbee2mqtt-3_backend if host_z2m3

    default_backend synology_backend
```

### Backends
```
backend glpi_backend
    mode http
    server glpi 192.168.112.30:28884 check

backend qiskit_backend
    mode http
    server qiskit 192.168.112.30:28888 check

backend zigbee2mqtt-1_backend
    mode http
    server z2m1 192.168.112.30:8100 check

backend zigbee2mqtt-2_backend
    mode http
    server z2m2 192.168.112.30:8101 check

backend zigbee2mqtt-3_backend
    mode http
    server z2m3 192.168.112.30:8102 check

backend synology_backend
    mode http
    server syno 192.168.118.2:443 ssl verify none check

backend letsencrypt_backend
    mode http
    server localhost 127.0.0.1:8800
```

---

## Zertifikate mit acme.sh erstellen

### Schritt 1: Zertifikat erzeugen
```
acme.sh --issue --standalone --httpport 8800 -d sub.domain.ltd
```

---

### Schritt 2: Zertifikat in HAProxy-Ordner installieren
```
acme.sh --install-cert -d sub.domain.ltd \
  --key-file /etc/haproxy/certs/sub.domain.ltd.key \
  --fullchain-file /etc/haproxy/certs/sub.domain.ltd.pem \
  --reloadcmd "systemctl reload haproxy"
```

### Schritt 3: Key und Zertifikat zu einer HAProxy-Datei zusammenführen
```
cd /etc/haproxy/certs
cat sub.domain.ltd.key sub.domain.ltd.pem >sub.domain.ltd.pem.tmp
mv sub.domain.ltd.pem.tmp sub.domain.ltd.pem
```

### HAProxy testen und neu laden
```
haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl reload haproxy
```

---

## Hinweise

- `/etc/haproxy/certs/` enthält für jede Domain eine Datei, die Key und Fullchain enthält.
- HAProxy lädt automatisch **alle** Zertifikate aus dem Ordner, wenn man `crt /etc/haproxy/certs/` nutzt.
- Achte darauf, dass keine `.fullchain.pem` ohne Key im Ordner liegt, sonst startet HAProxy nicht.


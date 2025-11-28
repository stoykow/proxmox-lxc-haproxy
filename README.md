# proxmox-lxc-haproxy
 Proxmox LXC HAProxy Reverse Proxy Setup mit Let's Encrypt (acme.sh) und Subdomain-Routing
Dieses Dokument beschreibt ein vollständiges Beispielsetup für einen HAProxy Reverse Proxy
in einer privaten Umgebung (Proxmox LXC). Es umfasst HTTP- und HTTPS-Routing, TLS-Terminierung
mit acme.sh (Let’s Encrypt), automatische Zertifikatverlängerung und die Weiterleitung
von Subdomains auf interne Dienste.

Alle Konfigurationen sind allgemein gehalten und enthalten keine sensiblen Daten.

---

## Inhalt

1. Übersicht
2. Architektur
3. Voraussetzungen
4. Domain-Konfiguration
5. LXC-Setup auf Proxmox
6. HAProxy Konfiguration
7. Let's Encrypt mit acme.sh
8. Automatische Zertifikat-Erneuerung
9. Routing-Beispiele
10. WebSockets Hinweis
11. Fehlerquellen
12. Lizenz

---

## 1. Übersicht

Ziele des Setups:

- zentraler Reverse Proxy auf Proxmox
- Routing von Subdomains auf interne Container oder Server
- TLS-Beendigung per Let’s Encrypt
- automatische Zertifikatverlängerung
- HTTPS-Passthrough für Legacy-Dienste
- Migration von externen Reverse Proxys

---

## 2. Architektur

Internet
↓
HAProxy Reverse Proxy
↓
Weiterleitung an interne Dienste

---

## 3. Voraussetzungen

- Proxmox VE
- LXC-Container mit Ubuntu 22.04
- Domain (Beispiel: domain.ltd)
- DNS-Zugriff (A-Record oder CNAME)
- Portweiterleitung 80/443 zur HAProxy-VM

---

## 4. Domain-Konfiguration

Hauptdomain:

domain.ltd     A      <öffentliche_IP>

Wildcard-Subdomain:

*.domain.ltd   CNAME  ddns.domain.ltd

Damit werden alle Subdomains an den Reverse Proxy weitergeleitet.

---

## 5. LXC-Container auf Proxmox

Ubuntu 22.04 erstellen.
Statische IP oder DHCP-Reservierung.

Pakete installieren:

apt update
apt install haproxy socat curl -y

---

## 6. HAProxy Konfiguration

Datei:

/etc/haproxy/haproxy.cfg

Beispiel:

frontend http_in
    bind *:80
    mode http

    acl letsencrypt_http path_beg /.well-known/acme-challenge/
    use_backend letsencrypt_backend if letsencrypt_http

    acl host_service1 hdr(host) -i service1.domain.ltd
    use_backend service1_backend if host_service1

    default_backend default_backend

backend service1_backend
    mode http
    server s1 192.168.0.10:8080 check

backend default_backend
    mode http
    server default 192.168.0.20:80 check

frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/
    mode tcp
    default_backend https_passthrough_backend

backend https_passthrough_backend
    mode tcp
    server passthrough 192.168.0.20:443 check

backend letsencrypt_backend
    mode http
    server localhost 127.0.0.1:8800

---

## 7. Let's Encrypt (acme.sh)

Installation:

curl https://get.acme.sh | sh
source ~/.bashrc

Zertifikat anfordern:

acme.sh --issue --standalone --httpport 8800 -d service1.domain.ltd

---

## 8. Automatische Zertifikat-Erneuerung

acme.sh --install-cert -d service1.domain.ltd \
--key-file /etc/haproxy/certs/service1.key \
--fullchain-file /etc/haproxy/certs/service1.pem \
--reloadcmd "systemctl reload haproxy"

acme.sh erzeugt automatisch einen Cronjob.

cd /etc/haproxy/certs
cat qiskit.domain.ltd.key qiskit.domain.ltd.pem > qiskit.domain.ltd.pem.tmp
mv qiskit.domain.ltd.pem.tmp qiskit.domain.ltd.pem

---

## 9. Routing-Beispiele

service1.domain.ltd → interner Dienst 1  
service2.domain.ltd → interner Dienst 2  
unbekannte Subdomains → Fallback Backend

---

## 10. WebSockets

WebSockets funktionieren in HAProxy normalerweise ohne Änderungen.

Bei Bedarf:

option http-buffer-request
option forwardfor

---

## 11. Fehlerquellen

- Port 80 für ACME-Challenge nicht erreichbar
- falscher Backend-Port
- Proxy-Chain blockiert HTTP Challenge
- HTTPS-Passthrough überschreibt Routing

---

## 12. Lizenz

MIT-Lizenz

Erlaubt Nutzung, Veränderung, Weitergabe.
Keine Haftung, keine Garantie.

---

Ende der Datei

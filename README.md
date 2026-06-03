# Tutorial: Desplegament d'una VPN amb WireGuard a AWS

**Requisits:** Compte AWS (Free Tier), màquina Kali Linux, connexió a internet

---

## Què construirem?

Un servidor VPN personal desplegat a AWS EC2 amb WireGuard. Un cop configurat, tot el tràfic de la teva màquina Kali passarà a través del servidor d'AWS, permetent-te inspeccionar paquets amb `tcpdump`, revisar logs i navegar per internet de forma segura.

---

## Part 1 — Configuració de l'instància EC2 a AWS

### Pas 1 — Llançar una instància EC2

1. Accedeix a **AWS Console → EC2 → Launch Instance**
2. Configura els següents paràmetres:
   - **Nom:** `vpn-server`
   - **AMI:** Ubuntu Server 24.04 LTS *(Free Tier eligible)*
   - **Tipus d'instància:** `t3.micro`
   - **Key pair:** Crea'n un de nou → descarrega el fitxer `.pem` i guarda'l en un lloc segur

3. A **Network Settings → Security Group**, afegeix aquestes regles d'entrada:

| Tipus | Protocol | Port | Origen |
|-------|----------|------|--------|
| SSH | TCP | 22 | La teva IP |
| Custom UDP | UDP | 51820 | 0.0.0.0/0 |

4. Fes clic a **Launch Instance**

### Pas 2 — Assignar una IP estàtica (Elastic IP)

Sense això, la IP pública del servidor canvia cada vegada que s'atura.

1. **EC2 → Elastic IPs → Allocate Elastic IP**
2. Associa-la a la teva instància

---

## Part 2 — Configuració del servidor WireGuard (EC2)

### Pas 3 — Connectar-se al servidor via SSH

```bash
chmod 400 la-teva-clau.pem
ssh -i la-teva-clau.pem ubuntu@<IP_PUBLICA_EC2>
```

### Pas 4 — Instal·lar WireGuard

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y wireguard wireguard-tools
```

### Pas 5 — Generar les claus del servidor

```bash
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
sudo chmod 600 /etc/wireguard/server_private.key

# Mostra les claus i guarda-les
sudo cat /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_public.key
```

### Pas 6 — Activar el reenviament d'IP

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Pas 7 — Crear la configuració del servidor

```bash
sudo nano /etc/wireguard/wg0.conf
```

Enganxa el següent contingut (substitueix `CLAU_PRIVADA_SERVIDOR` pel valor real):

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = CLAU_PRIVADA_SERVIDOR

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE

[Peer]
PublicKey = CLAU_PUBLICA_CLIENT
AllowedIPs = 10.8.0.2/32
```

> **Nota:** La interfície de xarxa a AWS sol ser `ens5`. Pots verificar-ho amb `ip route | grep default`.

---

## Part 3 — Configuració del client WireGuard (Kali)

### Pas 8 — Instal·lar WireGuard al client

```bash
sudo apt install -y wireguard wireguard-tools
```

### Pas 9 — Generar les claus del client

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key

# Mostra les claus
cat client_private.key
cat client_public.key
```

### Pas 10 — Trobar la porta d'enllaç local

```bash
ip route | grep default
```

Anota la IP que apareix després de `via` (per exemple `192.168.1.1`).

### Pas 11 — Crear el script vpn-route.sh

```bash
sudo nano /usr/local/bin/vpn-route.sh
```

```ìni
#!/bin/bash
GATEWAY=$(ip route | awk '/default/ {print $3; exit}')
EC2_IP="<IP_PUBLICA_EC2>"

if [ "$1" == "up" ]; then
    ip route add $EC2_IP/32 via $GATEWAY
    iptables -A OUTPUT -d $EC2_IP -p tcp --dport 22 -j ACCEPT
    iptables -A OUTPUT -d $EC2_IP -o wg0 -j DROP
elif [ "$1" == "down" ]; then
    ip route del $EC2_IP/32
    iptables -D OUTPUT -d $EC2_IP -p tcp --dport 22 -j ACCEPT
    iptables -D OUTPUT -d $EC2_IP -o wg0 -j DROP
fi
```
Converteix el fitxer en un executable

```bash
sudo chmod +x /usr/local/bin/vpn-route.sh
```

### Pas 12 — Crear la configuració del client

Crea la teva configuració de WireGuard com a client

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.8.0.2/24
PrivateKey = CLAU_PRIVADA_CLIENT
DNS = 1.1.1.1
PostUp = /usr/local/bin/vpn-route.sh up
PreDown = /usr/local/bin/vpn-route.sh down


[Peer]
PublicKey = CLAU_PUBLICA_SERVIDOR
Endpoint = <IP_PUBLICA_EC2>:51820
AllowedIPs = 0.0.0.0/1, 128.0.0.0/1
PersistentKeepalive = 25
```

Recorda substituir els valors entre `< >` pels teus valors reals.

---

## Part 4 — Iniciar Servidor

### Pas 13 — Iniciar WireGuard al servidor amb systemd

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show
```

Hauries de veure el peer del client llistat amb `allowed ips: 10.8.0.2/32`.

---

## Part 5 — Connexió i proves

### Pas 14 — Connectar la VPN des de Kali

> **Ordre important:** Primer obre el SSH, després activa la VPN.

**Terminal 1 — connecta't al servidor:**
```bash
ssh -i la-teva-clau.pem ubuntu@<IP_PUBLICA_EC2>
```

**Terminal 2 — activa la VPN:**
```bash
sudo wg-quick up wg0
```

### Pas 15 — Verificar que tot funciona

**Al client (Kali):**

```bash
# Comprova l'estat del túnel
sudo wg show

# Ping al servidor a través del túnel
ping 10.8.0.1

# Ping a internet
ping 8.8.8.8
ping google.com

# Comprova que la IP pública és la de l'EC2
curl ifconfig.me
```

**Al servidor (EC2) — observa el tràfic en temps real:**

```bash
sudo tcpdump -i wg0 -n
```

Hauràs de veure paquets com:
```
10.8.0.2 > 8.8.8.8: ICMP echo request
8.8.8.8 > 10.8.0.2: ICMP echo reply
```

---

## Part 6 — Gestió i logs

### Comandes útils al servidor

```bash
# Veure estat de WireGuard
sudo wg show

# Veure logs del servei
sudo journalctl -u wg-quick@wg0 -n 50

# Capturar tot el tràfic del túnel
sudo tcpdump -i wg0 -n

# Capturar només ICMP (pings)
sudo tcpdump -i wg0 -n icmp

# Capturar consultes DNS
sudo tcpdump -i wg0 -n port 53

# Reiniciar el servei
sudo systemctl restart wg-quick@wg0
```

### Comandes útils al client

```bash
# Activar VPN
sudo wg-quick up wg0

# Desactivar VPN
sudo wg-quick down wg0

# Veure estat
sudo wg show
```

---

## Resolució de problemes habituals (FAQ)

| Problema | Solució |
|----------|---------|
| `Permission denied` en generar claus | Afegeix `sudo` a la comanda |
| SSH es congela en activar la VPN | Obre el SSH **abans** d'activar la VPN |
| No hi ha internet però el ping al servidor funciona | Comprova que `DNS = 1.1.1.1` és al `wg0.conf` del client |
| `wg show` no mostra cap peer | Afegeix el peer al `wg0.conf` del servidor i reinicia |
| `resolvconf: command not found` | Executa `sudo apt install -y resolvconf` |
| La IP pública no és la de l'EC2 | Normal amb `AllowedIPs = 0.0.0.0/1, 128.0.0.0/1`; el tràfic igualment passa pel túnel |
| La VPN funciona a algunes xarxes i a unes altres no | Algunes xarxes (eduoram) bloquejen el trafic UDP. Canvia el ListeningPort de 51820 -> 443 tant al client com al servidor |

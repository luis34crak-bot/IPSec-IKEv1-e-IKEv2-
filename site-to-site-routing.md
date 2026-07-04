# 🔐 VPN Site-to-Site Basada en Enrutamiento (Route-Based / VTI)

## 1. Objetivo

Implementar una VPN site-to-site basada en enrutamiento usando **VTI (Virtual Tunnel Interface)**, con dos túneles:

| Túnel | Protocolo | Uso |
|-------|-----------|-----|
| **Tunnel1** | IKEv1 + IPsec | Túnel de respaldo |
| **Tunnel2** | IKEv2 + IPsec | Enrutamiento de tráfico entre LANs |

**Tráfico a comunicar:**

| Sede | Red LAN |
|------|---------|
| R1 | `7.84.0.0/24` |
| R2 | `7.84.1.0/24` |

---

## 2. Topología y Direccionamiento

```
                [ISP]
        e0/0            e0/1
     1.1.1.1/30      1.1.1.5/30
        │                 │
     e0/0 (WAN)       e0/0 (WAN)
   1.1.1.2/30        1.1.1.6/30
      [R1]              [R2]
   e0/1 (LAN)        e0/1 (LAN)
   7.84.0.1/24       7.84.1.1/24
      │                 │
    LAN R1            LAN R2
  7.84.0.0/24       7.84.1.0/24
```

### Direccionamiento de Túneles

| Túnel | Router | Dirección IP |
|-------|--------|--------------|
| Tunnel1 (IKEv1) | R1 | `10.1.1.1/30` |
| Tunnel1 (IKEv1) | R2 | `10.1.1.2/30` |
| Tunnel2 (IKEv2) | R1 | `10.2.2.1/30` |
| Tunnel2 (IKEv2) | R2 | `10.2.2.2/30` |

---

## 3. Configuración ISP (Núcleo)

> El ISP solo enruta tráfico público entre R1 y R2.

```
hostname ISP

interface Ethernet0/0
 ip address 1.1.1.1 255.255.255.252
 no shutdown

interface Ethernet0/1
 ip address 1.1.1.5 255.255.255.252
 no shutdown

! Rutas hacia las LANs (recomendadas para reachability y troubleshooting)
ip route 7.84.0.0 255.255.255.0 1.1.1.2
ip route 7.84.1.0 255.255.255.0 1.1.1.6
```

---

## 4. Configuración R1 (Sede Local)

### 4.1 Conectividad Base

```
hostname R1

interface Ethernet0/0
 description ENLACE_HACIA_ISP
 ip address 1.1.1.2 255.255.255.252
 no shutdown

interface Ethernet0/1
 description ENLACE_HACIA_LAN_R1
 ip address 7.84.0.1 255.255.255.0
 no shutdown

! Ruta por defecto hacia ISP
ip route 0.0.0.0 0.0.0.0 1.1.1.1
```

### 4.2 Tunnel1 = IKEv1 + IPsec (VTI)

**IKEv1 (Fase 1)**
```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key CISCO123 address 1.1.1.6
```

**IPsec (Fase 2)**
```
crypto ipsec transform-set TSET_V1 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROF_V1
 set transform-set TSET_V1
```

**Interfaz Tunnel1**
```
interface Tunnel1
 description VPN_IKEv1_HACIA_R2
 ip address 10.1.1.1 255.255.255.252
 tunnel source 1.1.1.2
 tunnel destination 1.1.1.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF_V1
```

### 4.3 Tunnel2 = IKEv2 + IPsec (VTI)

**IKEv2 (Proposal/Policy)**
```
crypto ikev2 proposal PROP_V2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL_V2
 proposal PROP_V2
```

**IKEv2 (Keyring/Profile)** — peer real = R2
```
crypto ikev2 keyring KEYS
 peer R2
  address 1.1.1.6
  pre-shared-key CISCO123

crypto ikev2 profile PROFILE_V2
 match identity remote address 1.1.1.6 255.255.255.255
 identity local address 1.1.1.2
 authentication remote pre-share
 authentication local pre-share
 keyring traditional KEYS
```

**IPsec Profile** (incluye transform-set + ikev2-profile)
```
crypto ipsec transform-set TSET_V2 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROF_V2
 set transform-set TSET_V2
 set ikev2-profile PROFILE_V2
```

**Interfaz Tunnel2**
```
interface Tunnel2
 description VPN_IKEv2_HACIA_R2
 ip address 10.2.2.1 255.255.255.252
 tunnel source 1.1.1.2
 tunnel destination 1.1.1.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF_V2
```

### 4.4 Enrutamiento de LAN por Tunnel2 (IKEv2)

```
ip route 7.84.1.0 255.255.255.0 Tunnel2
```

---

## 5. Configuración R2 (Sede Remota)

### 5.1 Conectividad Base

```
hostname R2

interface Ethernet0/0
 description ENLACE_HACIA_ISP
 ip address 1.1.1.6 255.255.255.252
 no shutdown

interface Ethernet0/1
 description ENLACE_HACIA_LAN_R2
 ip address 7.84.1.1 255.255.255.0
 no shutdown

! Ruta por defecto hacia ISP
ip route 0.0.0.0 0.0.0.0 1.1.1.5
```

### 5.2 Tunnel1 = IKEv1 + IPsec (VTI)

**IKEv1 (Fase 1)**
```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key CISCO123 address 1.1.1.2
```

**IPsec (Fase 2)**
```
crypto ipsec transform-set TSET_V1 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROF_V1
 set transform-set TSET_V1
```

**Interfaz Tunnel1**
```
interface Tunnel1
 description VPN_IKEv1_HACIA_R1
 ip address 10.1.1.2 255.255.255.252
 tunnel source 1.1.1.6
 tunnel destination 1.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF_V1
```

### 5.3 Tunnel2 = IKEv2 + IPsec (VTI)

**IKEv2 (Proposal/Policy)**
```
crypto ikev2 proposal PROP_V2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL_V2
 proposal PROP_V2
```

**IKEv2 (Keyring/Profile)** — peer real = R1
```
crypto ikev2 keyring KEYS
 peer R1
  address 1.1.1.2
  pre-shared-key CISCO123

crypto ikev2 profile PROFILE_V2
 match identity remote address 1.1.1.2 255.255.255.255
 identity local address 1.1.1.6
 authentication remote pre-share
 authentication local pre-share
 keyring traditional KEYS
```

**IPsec Profile** (incluye transform-set + ikev2-profile)
```
crypto ipsec transform-set TSET_V2 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile PROF_V2
 set transform-set TSET_V2
 set ikev2-profile PROFILE_V2
```

**Interfaz Tunnel2**
```
interface Tunnel2
 description VPN_IKEv2_HACIA_R1
 ip address 10.2.2.2 255.255.255.252
 tunnel source 1.1.1.6
 tunnel destination 1.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PROF_V2
```

### 5.4 Enrutamiento de LAN por Tunnel2 (IKEv2)

```
ip route 7.84.0.0 255.255.255.0 Tunnel2
```

---

## 6. Configuración PCs (VPCS)

| PC | Comando |
|----|---------|
| **PC1** | `ip 7.84.0.10 255.255.255.0 7.84.0.1` |
| **PC2** | `ip 7.84.1.10 255.255.255.0 7.84.1.1` |

---

## 7. Pruebas y Verificación

### ✅ Prueba "de oro"

Desde PC1:
```
ping 7.84.1.10
```

### 🔎 Verificación de IKE e IPsec

**IKEv1**
```
show crypto isakmp sa
show crypto ipsec sa
```

**IKEv2**
```
show crypto ikev2 sa
show crypto ipsec sa
```

**Cifrado en tiempo real**
```
show crypto engine connections active
```

---

## 8. Capturas de Pantalla Requeridas (con explicación)

- [ ] Topología completa con IPs
- [ ] `show ip interface brief` (ISP / R1 / R2)
- [ ] `show ip route` (ISP / R1 / R2) y rutas por Tunnel2
- [ ] `show run interface tunnel1` y `show run interface tunnel2`
- [ ] `show crypto isakmp sa` (IKEv1) y `show crypto ikev2 sa` (IKEv2)
- [ ] `show crypto ipsec sa` (contadores encaps/decaps)
- [ ] Ping PC1 → PC2 y salida de `show crypto engine connections active`

---

Sube esto en la parte de site to site en enrutamiento

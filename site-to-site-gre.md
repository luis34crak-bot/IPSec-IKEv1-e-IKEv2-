# 🔐 VPN Site-to-Site con Túnel GRE (GRE sobre IPsec)

Este escenario crea un túnel **GRE (Tunnel0)** entre R1 y R2 y lo protege con **IPsec**.
Incluye 2 opciones para la negociación: **IKEv1** o **IKEv2** (usa solo una a la vez).

---

## 1. Topología

### 🌐 WAN (público / ISP)

| Dispositivo | Interfaz | IP / Máscara | Hacia |
|-------------|----------|---------------|-------|
| R1 | e0/0 | `1.1.1.2/30` | ISP e0/0 (`1.1.1.1`) |
| R2 | e0/0 | `1.1.1.6/30` | ISP e0/1 (`1.1.1.5`) |

### 🏠 LAN (privado)

| Dispositivo | Interfaz | Red | Gateway |
|-------------|----------|-----|---------|
| R1 | e0/1 | `7.84.0.0/24` | `7.84.0.1` |
| R2 | e0/1 | `7.84.1.0/24` | `7.84.1.1` |

### 🌀 GRE (Tunnel0)

| Router | Tunnel0 IP |
|--------|------------|
| R1 | `10.0.0.1/30` |
| R2 | `10.0.0.2/30` |

**PSK (pre-shared-key):** `cisco123`

---

## 2. Reglas Importantes (para que no falle)

- GRE se protege con IPsec en **mode transport**.
- **IKEv1:** IPsec se aplica con `crypto map` en la WAN (e0/0), y la ACL cifra GRE.
- **IKEv2:** IPsec se aplica con `tunnel protection ipsec profile` directamente en Tunnel0.
- Enrutamiento LAN↔LAN se hace por rutas estáticas apuntando al next-hop del túnel (`10.0.0.1`/`10.0.0.2`).
- ⚠️ **No mezcles:** para pruebas, configura solo IKEv1 o solo IKEv2.

---

## 🅰️ Opción A — IKEv1 (Crypto Map en WAN) + GRE

### A.1 — R1 (IKEv1 + GRE)

```
! ======================
! IKEv1 - Fase 1 (ISAKMP)
! ======================
crypto isakmp policy 10
 encryption aes
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key cisco123 address 1.1.1.6

! ======================
! IPsec - Fase 2
! (Transport para GRE)
! ======================
crypto ipsec transform-set TS_GRE esp-aes esp-sha256-hmac
 mode transport

! ============================================
! Tráfico interesante: SOLO GRE entre las WANs
! ============================================
access-list 100 permit gre host 1.1.1.2 host 1.1.1.6

! =========
! Crypto Map
! =========
crypto map CMAP_GRE 10 ipsec-isakmp
 set peer 1.1.1.6
 set transform-set TS_GRE
 match address 100

! =========
! GRE Tunnel
! =========
interface Tunnel0
 description GRE_HACIA_R2 (protegido por IPsec IKEv1 en WAN)
 ip address 10.0.0.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 1.1.1.6

! =========================
! Aplicar IPsec en la WAN
! =========================
interface Ethernet0/0
 crypto map CMAP_GRE
```

### A.2 — R2 (IKEv1 + GRE)

```
crypto isakmp policy 10
 encryption aes
 hash sha256
 authentication pre-share
 group 14

crypto isakmp key cisco123 address 1.1.1.2

crypto ipsec transform-set TS_GRE esp-aes esp-sha256-hmac
 mode transport

! ACL espejo (en sentido contrario)
access-list 100 permit gre host 1.1.1.6 host 1.1.1.2

crypto map CMAP_GRE 10 ipsec-isakmp
 set peer 1.1.1.2
 set transform-set TS_GRE
 match address 100

interface Tunnel0
 description GRE_HACIA_R1 (protegido por IPsec IKEv1 en WAN)
 ip address 10.0.0.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 1.1.1.2

interface Ethernet0/0
 crypto map CMAP_GRE
```

### ✅ Verificación (IKEv1)

```
show crypto isakmp sa
show crypto ipsec sa
```

---

## 🅱️ Opción B — IKEv2 (IPsec Profile en Tunnel0) + GRE

### B.1 — R1 (IKEv2 + GRE protegido)

```
! ==========================
! IKEv2: Proposal & Policy
! ==========================
crypto ikev2 proposal PROP_V2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL_V2
 proposal PROP_V2

! =================
! IKEv2: Keyring
! =================
crypto ikev2 keyring KEYS_V2
 peer R2
  address 1.1.1.6
  pre-shared-key cisco123

! =================
! IKEv2: Profile
! =================
crypto ikev2 profile IKEV2_PROF
 match identity remote address 1.1.1.6 255.255.255.255
 identity local address 1.1.1.2
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS_V2

! ==========================
! IPsec: Transport para GRE
! ==========================
crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile IPSEC_PROF_V2
 set transform-set TS_GRE_V2
 set ikev2-profile IKEV2_PROF

! =========
! GRE Tunnel (con protección IPsec)
! =========
interface Tunnel0
 description GRE_HACIA_R2 (protegido por IPsec IKEv2)
 ip address 10.0.0.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 1.1.1.6
 tunnel protection ipsec profile IPSEC_PROF_V2
```

### B.2 — R2 (IKEv2 + GRE protegido)

```
crypto ikev2 proposal PROP_V2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL_V2
 proposal PROP_V2

crypto ikev2 keyring KEYS_V2
 peer R1
  address 1.1.1.2
  pre-shared-key cisco123

crypto ikev2 profile IKEV2_PROF
 match identity remote address 1.1.1.2 255.255.255.255
 identity local address 1.1.1.6
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS_V2

crypto ipsec transform-set TS_GRE_V2 esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile IPSEC_PROF_V2
 set transform-set TS_GRE_V2
 set ikev2-profile IKEV2_PROF

interface Tunnel0
 description GRE_HACIA_R1 (protegido por IPsec IKEv2)
 ip address 10.0.0.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 1.1.1.2
 tunnel protection ipsec profile IPSEC_PROF_V2
```

### ✅ Verificación (IKEv2)

```
show crypto ikev2 sa
show crypto ipsec sa
```

---

## 3. Enrutamiento (común para ambas opciones)

Para que PC1 (`7.84.0.0/24`) y PC2 (`7.84.1.0/24`) se vean por GRE, agrega rutas:

**R1**
```
ip route 7.84.1.0 255.255.255.0 10.0.0.2
```

**R2**
```
ip route 7.84.0.0 255.255.255.0 10.0.0.1
```

---

## 4. Pruebas

Desde PC1:
```
ping 7.84.1.10
```

Luego revisa cifrado en tiempo real:
```
show crypto engine connections active
```

---

## 5. Capturas Recomendadas (para tu informe)

- [ ] Topología completa con IPs
- [ ] `show ip interface brief` (R1 / R2 / ISP)
- [ ] `show run interface tunnel0` (R1 y R2)
- [ ] `show crypto isakmp sa` (si usaste IKEv1) o `show crypto ikev2 sa` (si usaste IKEv2)
- [ ] `show crypto ipsec sa` (contadores encaps/decaps)
- [ ] Ping PC1 ↔ PC2

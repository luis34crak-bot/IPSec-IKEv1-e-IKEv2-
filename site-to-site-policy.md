# 🔐 VPN Site-to-Site Basada en Políticas (Policy-Based)

## 1. Objetivo

Implementar una VPN site-to-site **policy-based** entre R1 (lado izquierdo) y R2 (lado derecho) para cifrar únicamente el tráfico "interesante" entre las LAN:

| Sede | Red LAN |
|------|---------|
| **LAN Izquierda** | `7.84.0.0/24` |
| **LAN Derecha** | `7.84.1.0/24` |

Se presentan dos escenarios (elige uno para tu laboratorio):

| Escenario | Protocolo |
|-----------|-----------|
| **A** | IKEv1 (ISAKMP) + IPsec (policy-based) |
| **B** | IKEv2 + IPsec (policy-based) |

> ⚠️ **Nota:** No se deben mezclar IKEv1 e IKEv2 entre routers. En cada escenario, ambos routers usan la misma versión.

---

## 2. Topología y Direccionamiento IP

```
                [ISP]
        e0/0            e0/1
     1.1.1.1/30      1.1.1.5/30
        │                 │
     e0/0 (WAN)       e0/0 (WAN)
   1.1.1.2/30        1.1.1.6/30
   [R1 - Izquierda] [R2 - Derecha]
   e0/1 (LAN)        e0/1 (LAN)
   7.84.0.1/24       7.84.1.1/24
      │                 │
   LAN Izquierda     LAN Derecha
  7.84.0.0/24       7.84.1.0/24
```

### PCs (ejemplo)

| PC | IP | Gateway |
|----|----|---------|
| **PC1** | `7.84.0.10/24` | `7.84.0.1` |
| **PC2** | `7.84.1.10/24` | `7.84.1.1` |

---

## 3. Parámetros Usados

### 🔑 IKEv1 (Fase 1)

| Parámetro | Valor |
|-----------|-------|
| Encryption | AES-256 |
| Hash | SHA-256 |
| Authentication | pre-shared-key |
| DH group | 14 |

### 🔑 IKEv2

| Parámetro | Valor |
|-----------|-------|
| Encryption | AES-CBC-256 |
| Integrity | SHA-256 |
| DH group | 14 |
| Authentication | pre-shared-key |

### 🔒 IPsec (Fase 2)

| Parámetro | Valor |
|-----------|-------|
| ESP Encryption | AES-256 |
| ESP Integrity | HMAC-SHA-256 |
| Mode | tunnel |

---

## 4. Routing Mínimo (Obligatorio)

> Sin conectividad IP entre peers (WAN) y entre LANs a través del ISP, IKE no negocia.

**R1**
```
ip route 7.84.1.0 255.255.255.0 1.1.1.1
```

**R2**
```
ip route 7.84.0.0 255.255.255.0 1.1.1.5
```

**ISP**
```
ip route 7.84.0.0 255.255.255.0 1.1.1.2
ip route 7.84.1.0 255.255.255.0 1.1.1.6
```

---

## 🅰️ Escenario A — Policy-Based VPN con IKEv1 (ISAKMP)

### A.1 — R1 (IKEv1)

```
! ---------- Fase 1 (IKEv1 / ISAKMP) ----------
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key CISCO123 address 1.1.1.6

! ---------- Fase 2 (IPsec) ----------
crypto ipsec transform-set TSET_V1 esp-aes 256 esp-sha256-hmac
 mode tunnel

! Tráfico interesante (LAN izq -> LAN der)
ip access-list extended ACL_VPN_V1
 permit ip 7.84.0.0 0.0.0.255 7.84.1.0 0.0.0.255

crypto map CMAP_V1 10 ipsec-isakmp
 set peer 1.1.1.6
 set transform-set TSET_V1
 match address ACL_VPN_V1

interface e0/0
 crypto map CMAP_V1
```

### A.2 — R2 (IKEv1)

```
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key CISCO123 address 1.1.1.2

crypto ipsec transform-set TSET_V1 esp-aes 256 esp-sha256-hmac
 mode tunnel

! Tráfico interesante (LAN der -> LAN izq)
ip access-list extended ACL_VPN_V1
 permit ip 7.84.1.0 0.0.0.255 7.84.0.0 0.0.0.255

crypto map CMAP_V1 10 ipsec-isakmp
 set peer 1.1.1.2
 set transform-set TSET_V1
 match address ACL_VPN_V1

interface e0/0
 crypto map CMAP_V1
```

### A.3 — Verificación (IKEv1)

| Fase | Comando |
|------|---------|
| Fase 1 | `show crypto isakmp sa` |
| Fase 2 | `show crypto ipsec sa` |

---

## 🅱️ Escenario B — Policy-Based VPN con IKEv2

### B.1 — R1 (IKEv2)

```
! ---------- IKEv2: Proposal/Policy ----------
crypto ikev2 proposal PROP_V2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL_V2
 proposal PROP_V2

! ---------- IKEv2: Keyring ----------
crypto ikev2 keyring K_RING_V2
 peer R2_PEER
  address 1.1.1.6
  pre-shared-key CISCO123

! ---------- IKEv2: Profile ----------
crypto ikev2 profile PROF_V2
 match identity remote address 1.1.1.6 255.255.255.255
 identity local address 1.1.1.2
 authentication remote pre-share
 authentication local pre-share
 keyring local K_RING_V2

! ---------- IPsec ----------
crypto ipsec transform-set TSET_V2 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended ACL_VPN_V2
 permit ip 7.84.0.0 0.0.0.255 7.84.1.0 0.0.0.255

crypto map CMAP_V2 10 ipsec-isakmp
 set peer 1.1.1.6
 set transform-set TSET_V2
 set ikev2-profile PROF_V2
 match address ACL_VPN_V2

interface e0/0
 crypto map CMAP_V2
```

### B.2 — R2 (IKEv2)

```
crypto ikev2 proposal PROP_V2
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL_V2
 proposal PROP_V2

crypto ikev2 keyring K_RING_V2
 peer R1_PEER
  address 1.1.1.2
  pre-shared-key CISCO123

crypto ikev2 profile PROF_V2
 match identity remote address 1.1.1.2 255.255.255.255
 identity local address 1.1.1.6
 authentication remote pre-share
 authentication local pre-share
 keyring local K_RING_V2

crypto ipsec transform-set TSET_V2 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended ACL_VPN_V2
 permit ip 7.84.1.0 0.0.0.255 7.84.0.0 0.0.0.255

crypto map CMAP_V2 10 ipsec-isakmp
 set peer 1.1.1.2
 set transform-set TSET_V2
 set ikev2-profile PROF_V2
 match address ACL_VPN_V2

interface e0/0
 crypto map CMAP_V2
```

### B.3 — Verificación (IKEv2)

| SA | Comando |
|----|---------|
| IKEv2 SA | `show crypto ikev2 sa` |
| IPsec SA | `show crypto ipsec sa` |

---

## 5. Troubleshooting Rápido

### 🚫 Si no levanta:

- Verifica reachability: `ping 1.1.1.6` desde R1 y `ping 1.1.1.2` desde R2.
- Asegura rutas en ISP hacia ambas LAN.
- Revisa que la PSK coincida y que la ACL sea espejo.

### 🧹 Limpieza (si cambiaste cosas):

```
clear crypto isakmp sa
clear crypto ikev2 sa
clear crypto ipsec sa
```

---

## 6. Checklist de Capturas de Pantalla

- [ ] Establecimiento exitoso del túnel
- [ ] Entradas de la tabla de enrutamiento confirmando las rutas de tráfico
- [ ] Conexiones crypto activas

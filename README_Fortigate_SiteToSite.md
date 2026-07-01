# VPN Site-to-Site — FortiGate

**Estudiante:** Euni | **Matrícula:** 2024-1185  
**Institución:** Instituto Tecnológico de las Américas (ITLA)  
**Asignatura:** Seguridad de Redes

---

## Objetivo

Configurar una VPN Site-to-Site entre dos FortiGates usando IPSec, permitiendo comunicación cifrada entre las redes LAN de dos sitios a través de un router ISP que simula internet. Se verifica la conectividad mediante traceroute entre las dos LANs.

---

## Topología

```
[PC1]──[SWA]──[FGT-A]──────[ISP]──────[FGT-B]──[SWB]──[PC2]
               Port2  Port1       Port1  Port2
               .1/26   .65/30     .70/30  .129/26
                        .66  ISP  .69
```

### Direccionamiento IP

| Dispositivo  | Interfaz | Dirección IP         | Descripción           |
|--------------|----------|----------------------|-----------------------|
| FortiGate-A  | port1    | 10.11.85.65/30       | WAN → ISP             |
| FortiGate-A  | port2    | 10.11.85.1/26        | LAN → SWA → PC1       |
| ISP          | e0/0     | 10.11.85.66/30       | Hacia FortiGate-A     |
| ISP          | e0/1     | 10.11.85.69/30       | Hacia FortiGate-B     |
| FortiGate-B  | port1    | 10.11.85.70/30       | WAN → ISP             |
| FortiGate-B  | port2    | 10.11.85.129/26      | LAN → SWB → PC2       |
| PC1          | e0       | 10.11.85.2/26        | GW: 10.11.85.1        |
| PC2          | e0       | 10.11.85.130/26      | GW: 10.11.85.129      |

---

## Parámetros VPN

### Phase 1 (IKE)

| Parámetro         | Valor         |
|-------------------|---------------|
| Versión IKE       | 1             |
| Modo              | Main          |
| Autenticación     | Pre-shared Key|
| PSK               | ITLA2024      |
| Cifrado           | AES256        |
| Hash              | SHA256        |
| Grupo DH          | 14            |
| Lifetime          | 86400 s       |

### Phase 2 (IPSec)

| Parámetro         | Valor         |
|-------------------|---------------|
| Cifrado           | AES256        |
| Hash              | SHA256        |
| PFS               | Habilitado    |
| Lifetime          | 43200 s       |
| Subred local A    | 10.11.85.0/26 |
| Subred remota A   | 10.11.85.128/26|
| Subred local B    | 10.11.85.128/26|
| Subred remota B   | 10.11.85.0/26 |

---

## Configuración

### ISP (Cisco IOU — CLI)

```
enable
configure terminal
hostname ISP

interface Ethernet0/0
 ip address 10.11.85.66 255.255.255.252
 no shutdown

interface Ethernet0/1
 ip address 10.11.85.69 255.255.255.252
 no shutdown

ip route 10.11.85.0   255.255.255.192 10.11.85.65
ip route 10.11.85.128 255.255.255.192 10.11.85.70

end
write memory
```

### FortiGate-A — Interfaces (CLI)

```
config system interface
    edit port1
        set mode static
        set ip 10.11.85.65 255.255.255.252
        set allowaccess ping https ssh http
    next
    edit port2
        set mode static
        set ip 10.11.85.1 255.255.255.192
        set allowaccess ping https ssh http
    next
end

config router static
    edit 1
        set gateway 10.11.85.66
        set device port1
    next
end
```

### FortiGate-B — Interfaces (CLI)

```
config system interface
    edit port1
        set mode static
        set ip 10.11.85.70 255.255.255.252
        set allowaccess ping https ssh http
    next
    edit port2
        set mode static
        set ip 10.11.85.129 255.255.255.192
        set allowaccess ping https ssh http
    next
end

config router static
    edit 1
        set gateway 10.11.85.69
        set device port1
    next
end
```

### FortiGate-A — VPN (GUI: VPN → IPsec Wizard)

| Campo            | Valor              |
|------------------|--------------------|
| Name             | VPN-A-a-B          |
| Template         | Site to Site       |
| Remote IP        | 10.11.85.70        |
| Interface        | port1              |
| Pre-shared Key   | ITLA2024           |
| Local interface  | port2              |
| Local subnet     | 10.11.85.0/26      |
| Remote subnet    | 10.11.85.128/26    |

### FortiGate-B — VPN (GUI: VPN → IPsec Wizard)

| Campo            | Valor              |
|------------------|--------------------|
| Name             | VPN-B-a-A          |
| Template         | Site to Site       |
| Remote IP        | 10.11.85.65        |
| Interface        | port1              |
| Pre-shared Key   | ITLA2024           |
| Local interface  | port2              |
| Local subnet     | 10.11.85.128/26    |
| Remote subnet    | 10.11.85.0/26      |

### VPCS

```
! PC1
ip 10.11.85.2 255.255.255.192 10.11.85.1
save

! PC2
ip 10.11.85.130 255.255.255.192 10.11.85.129
save
```

---

## Verificación

### En FortiGate CLI

```
get vpn ipsec tunnel summary
diagnose vpn ike gateway list
diagnose vpn tunnel list
```

### Desde PC1

```
ping 10.11.85.130
trace 10.11.85.130
```

### Desde PC2

```
ping 10.11.85.2
trace 10.11.85.2
```

**Resultado esperado del traceroute desde PC1:**
```
trace to 10.11.85.130
1  10.11.85.1    (FortiGate-A LAN)
2  10.11.85.130  (PC2 destino)
```
El tráfico va directo por el túnel sin mostrar hops del ISP, confirmando que la VPN está activa.

---

## Contramédidas

| Amenaza                    | Contramédida                                      |
|----------------------------|---------------------------------------------------|
| Interceptación de tráfico  | AES-256 cifra todo el tráfico entre sitios        |
| Modificación de paquetes   | SHA-256 verifica integridad                       |
| Acceso no autorizado       | PSK requerida + autenticación IKE Phase 1         |
| Replay attacks             | Anti-replay habilitado por defecto                |
| Forward secrecy            | PFS habilitado — claves nuevas en cada sesión     |

---

*Documento generado para fines académicos — ITLA 2024-1185*

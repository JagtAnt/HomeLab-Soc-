# Home Lab SOC: Despliegue de SIEM, Telemetría y Respuesta a Incidentes

## Resumen Ejecutivo
Este proyecto consiste en el diseño, implementación y operación de un laboratorio de Centro de Operaciones de Seguridad (SOC) a nivel local. El objetivo principal es simular un entorno corporativo para monitorizar puntos finales (endpoints), recolectar telemetría avanzada, detectar técnicas de persistencia y escalada de privilegios, y responder a incidentes simulados utilizando herramientas de código abierto estándar de la industria.

## Tecnologías y Herramientas Utilizadas
- **SIEM / XDR:** Wazuh v4.9 (Open Source HIDS/SIEM).
- **Endpoints (Víctimas):** Windows 10 Enterprise LTSC (`win10`), Windows 7.
- **Sistema Atacante / Administración:** Kali Linux.
- **Infraestructura:** Oracle VirtualBox, Redes NAT segmentadas.
- **Telemetría Avanzada:** Microsoft Sysmon v15.22 (System Monitor).

---

## Topología de Red
La infraestructura está segmentada en una red NAT aislada (`10.0.5.0/24`) para garantizar la seguridad y conectividad controlada durante las simulaciones.

>![Topología de Red - Home Lab SOC](</images/diagrama.jfif>)

| Dispositivo | Sistema Operativo | Rol | IP Asignada |
| :--- | :--- | :--- | :--- |
| **SIEM Server** | Ubuntu Server (`ubuntu-server`) | Analítica y Recolección de Logs | `10.0.5.6` |
| **Endpoint 1** | Windows 10 LTSC (`Agente-Win10`) | Víctima Principal / Telemetría | `10.0.5.8` |
| **Endpoint 2** | Windows 7 | Sistema Legado Vulnerable | `10.0.5.9` |
| **Atacante** | Kali Linux | Simulación de Adversarios | `10.0.5.7` |

---

## Fases del Proyecto y Evidencias

### Fase 1: Despliegue de la Infraestructura Central (SIEM)
Se configuró un servidor Ubuntu (`ubuntu-server`) para alojar la arquitectura central de Wazuh (`wazuh-indexer`, `wazuh-manager`, `filebeat` y `wazuh-dashboard`), generando los certificados de seguridad y habilitando el acceso web mediante HTTPS en el puerto `443`.

![Instalación del servidor Wazuh en Ubuntu Server](</images/Captura de pantalla (226).jpg>)

---

### Fase 2: Configuración de Endpoints y Telemetría Avanzada
Para obtener visibilidad profunda a nivel de host en el equipo Windows 10 (`win10`), se implementó Microsoft Sysmon junto con el agente oficial de Wazuh.

1. **Instalación de Sysmon:** Se desplegó `Sysmon64.exe` (v15.22) aplicando una plantilla de configuración personalizada (`sysmonconfig-export.xml`) para registrar creación de procesos, cambios en el sistema y eventos avanzados.

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

![Instalación y validación de Sysmon64 en Windows 10](</images/Captura de pantalla (227).png>)

2. **Despliegue y Conexión del Agente Wazuh:** Mediante PowerShell con privilegios de administrador, se descargó e instaló el paquete `wazuh-agent-4.9.2-1.msi`, vinculándolo a la dirección IP del Manager (`10.0.5.6`) con el identificador `Agente-Win10`. Posteriormente, se inició el servicio `WazuhSvc` y se comprobó la conectividad en red mediante ICMP (`ping`).

```powershell
Invoke-WebRequest -Uri [https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi) -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='10.0.5.6' WAZUH_AGENT_NAME='Agente-Win10'
NET START WazuhSvc
ping 10.0.5.6
```

![Instalación del agente Wazuh y prueba de conectividad](</images/Captura de pantalla (229).png>)

3. **Verificación en el Dashboard:** En la consola central de Wazuh (`https://10.0.5.6`), se confirmó el estado **Active (1)** del endpoint y la recepción continua de telemetría y alertas de seguridad.

![Panel principal de Wazuh mostrando el agente activo y resumen de alertas](</images/Captura de pantalla (232).png>)

---

### Fase 3: Simulación de Ataques (Red Teaming / Post-Explotación)
Se ejecutó una simulación de compromiso en el endpoint Windows 10 orientada a establecer **Persistencia** y realizar una **Escalada de Privilegios** a nivel local.

#### Ataque: Creación de Cuenta Backdoor y Escalada a Administradores
- **Descripción:** Simulando que un atacante obtuvo acceso a una consola con privilegios en el endpoint, se creó una cuenta de usuario no autorizada llamada `HackerNinja` y se agregó inmediatamente al grupo local `Administradores` para asegurar el control total del sistema.
- **Comandos ejecutados en PowerShell:**

```powershell
net user HackerNinja P4ssw0rd!123 /add
net localgroup Administradores HackerNinja /add
```

![Ejecución de comandos para creación de usuario y escalada de privilegios](</images/Captura de pantalla (231).png>)

---

### Fase 4: Detección y Análisis Forense (Blue Teaming)
A través del módulo **Threat Hunting** del SIEM, se analizaron en tiempo real las alertas generadas por las acciones ofensivas en `Agente-Win10`.

#### 1. Correlación de Eventos en Threat Hunting
El motor de reglas de Wazuh detectó la cadena completa de actividad sospechosa a las `21:36`:
- **Regla `60109` (Nivel 8 - Media):** *User account enabled or created* (Creación y habilitación del usuario malicioso).
- **Regla `60110` (Nivel 8 - Media):** *User account changed* (Modificación de atributos de la cuenta).
- **Regla `60154` (Nivel 12 - Alta):** *Administrators Group Changed* (Inclusión del usuario en el grupo privilegiado de Administradores).

![Vista general de alertas detectadas en el módulo Threat Hunting](</images/Captura de pantalla (236).png>)

#### 2. Análisis del Log (Document Details / Raw Data)
Al inspeccionar el detalle del evento en Wazuh, se extrajeron los siguientes artefactos forenses clave:
- **Agente / Host afectado:** `Agente-Win10` (`win10`).
- **Canal y Event ID de Windows:** Canal `Security` — **Event ID `4722`** (Se habilitó una cuenta de usuario).
- **Cuenta origen (Sujeto):** `vboxuser` (Sesión `0x6B3FA`).
- **Cuenta creada (Objetivo):** `HackerNinja` (SID: `S-1-5-21-1509269586-3461795538-1266004415-1001`).
- **Decodificador y Grupos:** `windows_eventchannel` | Grupos: `windows, windows_security, adduser, account_changed`.
- **Tácticas de MITRE ATT&CK asociadas:** *Persistence* (T1136.001 - Create Account: Local Account) y *Privilege Escalation* (T1098 - Account Manipulation).
- **Mapeo de Cumplimiento Normativo:** GDPR (`IV_35.7.d`, `IV_32.2`), HIPAA (`164.312.a.2.I`, `164.312.a.2.II`, `164.312.b`) y GPG13 (`7.10`).

![Detalles del evento mostrando el usuario HackerNinja y el Event ID 4722](</images/Captura de pantalla (237).png>)

![Detalles de la regla 60109, decodificador y normativas de cumplimiento](</images/Captura de pantalla (238).png>)

---

### Fase 5: Respuesta a Incidentes y Remediación
Con base en los Indicadores de Compromiso (IOCs) obtenidos en el SIEM, se establecen las siguientes acciones de contención y erradicación:
1. **Aislamiento del Endpoint:** Desconectar temporalmente el equipo `win10` (`Agente-Win10`) de la red `10.0.5.0/24` para prevenir movimientos laterales.
2. **Revocación de Accesos No Autorizados:** Eliminar de inmediato la cuenta maliciosa creada mediante el comando `net user HackerNinja /delete` y verificar los miembros actuales del grupo `Administradores`.
3. **Auditoría de Cuenta Comprometida:** Investigar la actividad de la cuenta origen `vboxuser` (ID de inicio de sesión `0x6B3FA`) y rotar sus credenciales.
4. **Respuesta Activa (Active Response):** Configurar un script de respuesta activa en Wazuh para bloquear o revertir automáticamente la creación de usuarios locales no autorizados ante la activación de las reglas `60109` y `60154`.

---
*Proyecto documentado por: José A. Gómez - Creado en 2026*
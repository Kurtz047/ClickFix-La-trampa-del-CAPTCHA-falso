# ClickFix-La-trampa-del-CAPTCHA-falso
Bienvenidos al SOC. Hemos recibido una solicitud urgente de Recursos Humanos: la empleada Anita García informa que su estación de trabajo corp-wks-102 ha estado presentando un comportamiento extraño desde ayer por la tarde. Menciona haber visitado un sitio web de investigación que le solicitó completar una verificación CAPTCHA. 

# ¿Qué es ClickFix?
ClickFix es una técnica de ingeniería social que se popularizó durante 2024 y 2025. El atacante crea una página que simula un CAPTCHA o un paso de verificación rutinario y le pide al visitante que realice una breve secuencia de teclas: Win+R, luego Ctrl+V y finalmente Enter. No se ofrece ningún archivo ni se descarga nada, por lo que no hay ningún archivo adjunto que analizar ni ningún ejecutable que el navegador pueda bloquear. El código JavaScript de la página prepara en segundo plano un comando de PowerShell proporcionado por el atacante, y las teclas que pulsa el visitante son las que lo ejecutan.

# Lo que sabemos hasta ahora
Usuario afectado: anita.garcia (estación de trabajo corp-wks-102, IP 10.0.1.135)
Visitó eemmbryequo.shop, que mostraba una página CAPTCHA falsa.
La página le indicaba que presionara Win+R, pegara y pulsara Enter.
Se ejecutó un comando de PowerShell, descargando una carga útil desde promptcraft.online.
Nuestro agente XDR ha detectado múltiples procesos sospechosos.

Telemetría del punto final desde el momento de la ejecución
Antes de analizar el malware, veamos qué registró el agente de seguridad mientras Anita estaba en esa página. Dos de estas líneas corresponden a la navegación habitual. Las otras dos corresponden a la transferencia de control: la página interactuando con su equipo y las pulsaciones de teclas que la convirtieron en un proceso en ejecución.

<img width="1277" height="510" alt="Captura de pantalla 2026-11-02 132028" src="https://github.com/user-attachments/assets/4bb03690-97fb-4ef0-84fe-90119b6c6d6a" />

Por qué esto es importante
Todos los controles de esa estación de trabajo se diseñaron para inspeccionar los elementos que llegan: un archivo adjunto, una descarga, un instalador. ClickFix llega como texto y lo ejecuta el usuario, por lo que no hay nada que esos controles deban evaluar ni poner en cuarentena. El nombre del mecanismo indica dónde debe activarse la detección: en el momento en que aparece una consola iniciada por el usuario debajo del proceso de escritorio, en lugar de en el momento en que un archivo se guarda en el disco.

# Registro de Incidente SOC
## Información del caso
- **ID del caso:** SOC-2026-001
- **Fecha:** 30/10/2026
- **Analista:** Javier Sosa
- **Plataforma:** SOCSimulator
- **Severidad:** Alta
- **Estado:** Escalado a L2

## Resumen ejecutivo
Se detectó un intento de robo de credenciales (credential harvesting) mediante un correo de phishing que redirigía a un formulario de login falso de Microsoft. La usuaria afectada ingresó sus credenciales en el sitio fraudulento. Se identificó el dominio externo que recibió los datos y se recomendó el bloqueo y el reseteo de contraseña.

## Alerta inicial
- **Fuente:** Correo electrónico reportado por el usuario / SIEM / EDR
- **Descripción:** Correo con enlace a un sitio que imita el login de Microsoft.
- **Usuario afectado:** alicia.gomez@empresa.com
- **Workstation:** WS-ALICIA-01 (192.168.1.45)

## Investigación (L1)
1. **Análisis del correo:** El remitente utilizaba un dominio lookalike (ej: `micros0ft-support.com`). El enlace redirigía a un primer dominio legítimo comprometido y luego a un segundo dominio falso.
2. **Análisis de red:** Se revisaron los logs de la workstation de la usuaria. Se identificó una petición **POST** hacia el dominio `login-microsoft-alerta.com` a las 14:32 hs.
3. **Correlación:** El dominio falso imita la apariencia de Microsoft y recibió las credenciales ingresadas por la usuaria.

## Evidencia (IOCs)
- **Dominio falso:** `login-microsoft-alerta.com`
- **IP del servidor:** 203.0.113.45
- **Hash del archivo adjunto (si aplica):** `d41d8cd98f00b204e9800998ecf8427e`
- **URL completa:** `https://login-microsoft-alerta.com/auth/login`
- **Timestamp del POST:** 2026-10-30 14:32:10 UTC

## Acciones tomadas
- Se documentó el caso con línea de tiempo y evidencia.
- Se escaló a L2 por tratarse de credenciales comprometidas.
- Se recomendó:
  - Bloquear el dominio `login-microsoft-alerta.com` en el proxy/firewall.
  - Forzar reseteo de contraseña de la usuaria afectada.
  - Revisar si otros usuarios accedieron al mismo dominio.

## Lecciones aprendidas / Notas para el equipo
- El correo evadió el filtro de spam porque el primer salto era un dominio legítimo comprometido.
- Se sugiere capacitación al usuario sobre verificación de dominios en enlaces.
- Se recomienda agregar el dominio falso a la lista de bloqueo de correo.

## Clasificación final
- **Tipo de incidente:** Phishing / Credential Harvesting
- **MITRE ATT&CK:** T1566 (Phishing), T1078 (Valid Accounts)
- **Confianza:** Alta
- **Escalado a:** L2 (Investigación profunda)

# ClickFix-La-trampa-del-CAPTCHA-falso
Bienvenidos al SOC. Hemos recibido una solicitud urgente de Recursos Humanos: la empleada Anita García informa que su estación de trabajo corp-wks-102 ha estado presentando un comportamiento extraño desde ayer por la tarde. Menciona haber visitado un sitio web de investigación que le solicitó completar una verificación CAPTCHA. 
# Registro de Incidente SOC

## Información del caso
- **ID del caso:** SOC-2026-001
- **Fecha:** 30/10/2026
- **Analista:** Javier Sosa
- **Plataforma:** SOCSimulator / LetsDefend / TryHackMe (según corresponda)
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

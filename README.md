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

# Telemetría del punto final desde el momento de la ejecución
Antes de analizar el malware, veamos qué registró el agente de seguridad mientras Anita estaba en esa página. Dos de estas líneas corresponden a la navegación habitual. Las otras dos corresponden a la transferencia de control: la página interactuando con su equipo y las pulsaciones de teclas que la convirtieron en un proceso en ejecución.

<img width="1277" height="510" alt="Captura de pantalla 2026-11-02 132028" src="https://github.com/user-attachments/assets/4bb03690-97fb-4ef0-84fe-90119b6c6d6a" />

# Por qué esto es importante
Todos los controles de esa estación de trabajo se diseñaron para inspeccionar los elementos que llegan: un archivo adjunto, una descarga, un instalador. ClickFix llega como texto y lo ejecuta el usuario, por lo que no hay nada que esos controles deban evaluar ni poner en cuarentena. El nombre del mecanismo indica dónde debe activarse la detección: en el momento en que aparece una consola iniciada por el usuario debajo del proceso de escritorio, en lugar de en el momento en que un archivo se guarda en el disco.

Tras la ejecución inicial de PowerShell, nuestro agente XDR registró varias actividades del sistema de archivos en la estación de trabajo de Anita. La línea de código de PowerShell copiada extrajo un pequeño script de descarga (l6E.ps1) del servidor del atacante, que a su vez obtuvo y descargó la carga útil l6E.exe. Posteriormente, mshta.exe (una utilidad legítima de Windows que suele ser utilizada indebidamente por los atacantes) la ejecutó, y regasm.exe se utilizó para cargar una DLL maliciosa en la memoria

(Los binarios que se ejecutan fuera del entorno (LOLBins), como mshta.exe y regasm.exe, son herramientas legítimas de Windows que los atacantes utilizan indebidamente para ejecutar código malicioso. Dado que son binarios firmados por Microsoft, suelen eludir las listas blancas de aplicaciones y la detección de puntos finales.)

# Artefactos del sistema de archivos XDR
Analicemos los archivos registrados en corp-wks-102. Uno de ellos es la DLL maliciosa que cargó regasm.exe. ¿Puedes identificar el archivo de biblioteca sospechoso?

<img width="1279" height="516" alt="Captura de pantalla 2026-11-02 133308" src="https://github.com/user-attachments/assets/37950626-a8b0-4edb-8cec-1c9f4872bec7" />

El nombre de un archivo es una vulneración que el atacante puede aprovechar. Windows cargará una biblioteca con cualquier nombre, y uno que parezca familiar no le cuesta nada al atacante. La clave para resolver esto reside en lo que el agente del punto final registró de forma independiente sobre cada archivo, en lugar del nombre que el archivo mismo da. Introduzca el nombre del archivo exactamente como aparece en el registro XDR.

Por qué esto es importante
Una biblioteca sin firmar en un directorio de trabajo no es una prueba irrefutable por sí sola, y tratarla como tal es como los analistas pierden una tarde entera analizando archivos temporales de Office. El juicio que acabas de hacer es el que tiene mayor impacto: un archivo genera sospechas por la combinación de dónde se escribió, quién lo avaló y qué lo cargó. El nombre es el más débil de los tres, porque es la única parte que el atacante controla libremente.

Introduzca el nombre del archivo DLL malicioso que se cargó durante este ataque:
Microsoft.Runtime.dll

# Rastrea la descarga de la carga útil hasta su dirección IP de origen.
Ahora que hemos identificado la biblioteca maliciosa, analicemos cómo llegó al sistema. El comando que la víctima pegó ejecutó una línea de PowerShell que descargó un script de descarga (l6E.ps1) del servidor del atacante, el cual a su vez descargó e instaló la carga útil l6E.exe. El dominio involucrado fue promptcraft.online. Como analistas del SOC, necesitamos más que un nombre de dominio para poder configurar un bloqueo en el firewall o buscar otros hosts afectado

<img width="1277" height="520" alt="Captura de pantalla 2026-11-02 134030" src="https://github.com/user-attachments/assets/fa3f5697-a1ae-4a63-b944-4252488bf55f" />

Los dominios son baratos y desechables. Un operador registra uno nuevo en minutos, y el mismo proveedor de alojamiento puede gestionar fácilmente una docena de ellos, por lo que una lista de bloqueo basada únicamente en nombres de dominio queda obsoleta. La infraestructura es la parte más lenta de la pareja, así que merece la pena el esfuerzo adicional de vincular el nombre con la entidad que lo respalda. Responda con una única dirección IPv4.

Por qué esto es importante
Bloquear el dominio habría dado la sensación de ser una solución y habría dado aproximadamente un día de margen. La dirección es lo que permite plantear la pregunta realmente importante: qué otros hosts de la red se han comunicado con este dominio y desde cuándo. Esta consulta solo se ejecuta sobre la infraestructura, por lo que convertir un nombre en una dirección es lo primero que se hace con cualquier indicador de entrega, antes de la contención y mucho antes del informe.

¿Cuál es la dirección IP del servidor que alojó la carga útil maliciosa (promptcraft.online)?: 
185.91.69.119

#Desenmascarar el destino de comando y control
Excelente trabajo al identificar el origen del malware. Ahora investiguemos qué sucedió después de su instalación. El componente de robo de credenciales, cleanuploader.exe, accedió al almacén de credenciales de Chrome de Anita. Pero, ¿a dónde envió los datos robados? Los atacantes abusan cada vez más de los servicios legítimos en la nube para usarlos como servidor de comando y control (C2) y camuflarse con el tráfico normal.

Cadena de ejecución de procesos
Primero, examinemos el árbol de procesos de XDR para comprender cómo se relacionan entre sí los componentes del malware. Observe la cadena: powershell.exe inició mshta.exe, que usó regasm.exe para cargar la DLL, la cual finalmente generó el programa de robo de credenciales.

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

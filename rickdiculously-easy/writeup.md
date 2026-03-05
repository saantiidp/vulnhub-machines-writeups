# Rickdiculously Easy (VulnHub) en VMware – Instalación, enumeración y explotación (WriteUp)

> **Aviso importante (ético y legal):** Este documento está escrito **exclusivamente** con fines de aprendizaje en un **laboratorio controlado**.  
> La máquina *Rickdiculously Easy* está diseñada para ser vulnerable a propósito. **No** apliques estas técnicas sobre sistemas reales sin autorización explícita.

---

## Introducción

En este laboratorio trabajamos con **Rickdiculously Easy**, una máquina vulnerable estilo CTF de VulnHub (temática Rick & Morty) orientada a practicar un flujo completo de **pentesting**:

- Preparación del entorno en **VMware** (red aislada).
- Descubrimiento de IP y verificación de conectividad.
- Enumeración de puertos/servicios con **Nmap** en fases (metodología realista).
- Enumeración y explotación de **FTP anónimo**.
- Enumeración web (80/tcp): `robots.txt`, listing de directorios, y fuzzing con **dirsearch**.
- Detección de **Command Injection** en CGI (`tracertool.cgi`) y enumeración de usuarios locales.
- Pivot hacia un servicio **SSH alternativo** (22222/tcp) + reutilización de credenciales.
- Post-explotación inicial: enumeración, flags y recolección de artefactos (ZIP + JPG).
- Extracción de información oculta en imagen (`strings` / esteganografía “básica”).

> Este write-up documenta **todo lo realizado** hasta el punto actual. Iremos ampliándolo con tus siguientes pasos, sin omitir nada.

---

## Entorno de laboratorio

### Máquinas usadas

- **Atacante:** Kali Linux
- **Víctima:** Rickdiculously Easy (Fedora 26 Server)

### Red (muy importante)

Para que ambas máquinas se “vean”, se configuró una **red virtual aislada** en VMware dentro del rango:

- `192.168.247.0/24`

Esto es típico en auditorías en laboratorio, porque facilita:
- descubrimiento por ARP (`netdiscover`)
- conectividad simple y estable
- aislamiento del resto de tu red real

> Nota: En algunos entornos cloud como AWS, el descubrimiento ARP no funciona igual (o está mitigado) porque el proveedor controla/filtra ciertos comportamientos de red a nivel virtualización.

---

## 1. Arranque de la VM víctima y consola inicial

Una vez instalada e iniciada la VM, la consola muestra Fedora Server y un mensaje importante: existe un **Admin Console** accesible vía web en el puerto **9090** sobre localhost.

📷 **Imagen 1 — Arranque y consola de login**
![Boot / login](images/01-vm-boot-login.png)

### ¿Qué significa “Admin Console: https://127.0.0.1:9090”?

En Fedora/RedHat es común el servicio **Cockpit**, un panel web de administración del servidor.
- Suele correr en el puerto **9090**
- Por defecto, aquí aparece ligado a `localhost` (acceso local), pero en CTFs muchas veces está **expuesto en red**.

Esto se volverá relevante cuando enumeremos puertos abiertos.

---

## 2. Descubrimiento de la IP del objetivo con Netdiscover

Como no tenemos información inicial (ni IP ni hostname), el primer paso práctico es descubrir hosts activos en la red.

### 2.1 `netdiscover` (modo básico)

Desde Kali:

```bash
sudo netdiscover
```

#### ¿Qué hace netdiscover?
- Es una herramienta de **reconocimiento ARP**.
- Envía peticiones ARP en la red local y observa respuestas.
- Permite ver:
  - IPs activas
  - MAC addresses
  - Vendor (por ejemplo VMware)

📷 **Imagen 2 — Salida de netdiscover + ifconfig**
![netdiscover + ifconfig](images/02-netdiscover-ifconfig.png)

---

## 3. Determinar tu rango de red con `ifconfig`

Antes de escanear un rango con precisión, conviene conocer tu IP local y la máscara.

```bash
ifconfig
```

Salida relevante (Kali):

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.184.128  netmask 255.255.255.0  broadcast 192.168.184.255
```

### ¿Cómo saco el rango a partir de esto?

- IP: `192.168.184.128`
- Netmask: `255.255.255.0`  → equivale a `/24`

Por tanto, el rango es:

- `192.168.184.0/24`

---

## 4. Ajuste de red en VMware (red aislada 192.168.247.0/24)

Para este laboratorio se creó una red virtual nueva en VMware (por ejemplo “Bowling Alley / Parque de bolos”) con rango:

- `192.168.247.0/24`

Se asigna esa red tanto a:
- Kali (atacante)
- Rickdiculously Easy (víctima)

Así garantizamos que:
- Están en la misma LAN
- ARP funciona
- Netdiscover detecta hosts de forma fiable

---

## 5. Netdiscover con rango específico

`netdiscover` permite especificar el rango manualmente.

Primero, revisamos ayuda (para entender flags):

```bash
netdiscover --help
```

Fragmento clave:

```text
-i device: your network device
-r range: scan a given range instead of auto scan. 192.168.6.0/24,/16,/8
-p passive mode: do not send anything, only sniff
-f enable fastmode scan
```

### 5.1 Escaneo activo del rango

```bash
sudo netdiscover -r 192.168.247.0/24
```

### 5.2 Confirmación de “quién soy yo” (IP de Kali)

```bash
ifconfig
```

En este caso Kali queda en:

- `192.168.247.129`

### 5.3 Interpretación de hosts encontrados

En el resultado aparecen típicamente:

- `192.168.247.129` → Kali (tu máquina)
- `192.168.247.1` → Gateway virtual de VMware
- `192.168.247.254` → DHCP de VMware
- `192.168.247.128` → Otra VM (probable víctima)

Si en esa red **solo** hay dos máquinas, lo lógico es:

✅ `192.168.247.129` → Kali  
✅ `192.168.247.128` → Rickdiculously Easy

> Aun así, siempre se confirma con un escaneo de puertos (Nmap).

---

## 6. Confirmación del objetivo con Nmap (escaneo completo agresivo en lab)

Para confirmar que esa IP es la víctima, se lanza un escaneo completo:

```bash
nmap -p- -sCV -n -Pn -vvv --open -T5 -oN Rickdiculously 192.168.247.128
```

### Explicación de cada opción

- `-p-`  
  Escanea **todos** los puertos TCP (1–65535).  
  Útil en laboratorio porque no queremos perdernos servicios “raros” en puertos altos.

- `-sC`  
  Ejecuta scripts NSE “por defecto” (detección básica de info, banners, configuraciones típicas).

- `-sV`  
  Intenta detectar la **versión** de cada servicio.

- `-n`  
  No resuelve DNS (más rápido y menos ruido).

- `-Pn`  
  Asume que el host está activo (no hace ping discovery).  
  Útil cuando ICMP está filtrado o el host no responde a ping.

- `-vvv`  
  Muy verboso (ves avances y hallazgos en tiempo real).

- `--open`  
  Solo muestra puertos en estado `open` (salida más limpia).

- `-T5`  
  Timing agresivo (rápido, pero genera más ruido).  
  En laboratorio está bien; en real puede ser demasiado evidente.

- `-oN Rickdiculously`  
  Guarda la salida en un archivo “normal” para documentar.

### Puertos descubiertos (resumen)

```text
22/tcp
80/tcp
21/tcp
60000/tcp
13337/tcp
22222/tcp
9090/tcp
```

Esto ya te dice algo importante:

👉 **Máquina intencionalmente vulnerable** con múltiples superficies de ataque.

---

## 7. Metodología realista: Nmap en fases

En auditorías reales, normalmente se divide Nmap en varias fases para **reducir ruido**:

### Fase 1 — Descubrir puertos “con mínimo ruido”
Ejemplo (SYN scan, timing lento):

```bash
nmap 192.168.247.128 -sS -T0
```

- `-sS` (SYN scan):  
  Escaneo semi-sigiloso (no completa el handshake TCP completo).
- `-T0`:  
  Timing **más lento** (menos agresivo, menos sospechoso, pero puede tardar muchísimo).

> En un entorno real, **esto** es lo que harías primero para no “cantar” en logs/IDS.

### Fase 2 — Escaneo dirigido a los puertos encontrados
Cuando ya tienes una lista de puertos, haces enumeración de versiones/scripts solo ahí:

```bash
nmap 192.168.247.128 -p22,21,80,9090,22222 -sCV
```

Ventaja:
- Muchísimo menos ruido que `-p-` con scripts
- Mucho más realista

### Fase 3 — Enumeración específica por servicio
Después ya usas herramientas “especializadas”:
- FTP: `ftp`, `lftp`, `nmap --script ftp-*`
- SSH: `ssh`, `hydra`, `ssh-audit`
- Web: `dirsearch`, `ffuf`, `nikto`, `whatweb`, etc.

---

## 8. Enumeración y explotación inicial: FTP (21/tcp)

Nmap reporta:

```text
21/tcp open ftp vsftpd 3.0.3
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

### ¿Por qué es grave “Anonymous FTP”?

Porque permite conectarte como `anonymous` sin credenciales reales, y a veces:
- listar archivos
- descargarlos
- (peor) subir archivos si el servidor lo permite

### 8.1 Conexión FTP anónima

```bash
ftp -a 192.168.247.128
```

- `-a` intenta login anónimo automáticamente.

Salida:

```text
230 Login successful.
```

### 8.2 Listado y descarga de FLAG.txt

Dentro del prompt FTP:

```text
ftp> ls
```

Se observa:

- `FLAG.txt`
- directorio `pub`

Descarga:

```text
ftp> get FLAG.txt
```

Verificación en Kali:

```bash
cat FLAG.txt
```

Resultado:

```text
FLAG{Whoa this is unexpected} - 10 Points
```

### 8.3 Revisión del directorio `/pub`

```text
ftp> cd pub
ftp> ls
```

Está vacío.

✅ Pregunta correcta (mentalidad pro):  
**“¿Esto se refleja en web? ¿Existe una ruta web que apunte a /var/ftp o similar?”**

En este punto **no lo sabemos**, pero es un vector típico:
- FTP permite subir → la web ejecuta/expone → posible RCE (subida de webshell / reverse shell)

---

## 9. Enumeración web en 80/tcp (metodología)

En auditorías web solemos seguir un flujo bastante estándar:

1. Ver la web en navegador
2. Identificar tecnologías (headers, CMS, rutas típicas)
3. Enumerar directorios (fuzzing / fuerza bruta)
4. Revisar `robots.txt`, `sitemap.xml`
5. Buscar puntos de entrada (formularios, parámetros, uploads)
6. Probar vulnerabilidades (según hallazgos)

### 9.1 Diccionarios: SecLists (opcional)

```bash
sudo git clone https://github.com/danielmiessler/SecLists.git
```

Pero a veces tarda, así que usaste una herramienta directa:

### 9.2 Enumeración de rutas con `dirsearch`

```bash
dirsearch -u http://192.168.247.128
```

Resultados relevantes:

- `/passwords/` (200)
- `/robots.txt` (200)
- `/cgi-bin/` (403)

#### Interpretación
- `200` = recurso accesible
- `301` = redirect
- `403` = existe, pero no accesible directamente (aun así es una pista)

---

## 10. Ruta `/passwords/` y Directory Listing

Al abrir:

```text
http://192.168.247.128/passwords/
```

se observa un **Index of** (Directory Listing).

📷 **Imagen 3 — Directory Listing de /passwords/**
![Index /passwords](images/03-web-passwords-index.png)

Allí aparecen:
- `FLAG.txt`
- `passwords.html`

### 10.1 Flag en web

Al abrir `FLAG.txt`:

```text
FLAG{Yeah d- just don't do it.} - 10 Points
```

### 10.2 Fuente de `passwords.html` y contraseña escondida

En CTF y en el mundo real, **revisar el código fuente** es obligatorio.

Opciones:
- Click derecho → “View Page Source”
- o `view-source:` en el navegador

En el HTML se encontró un comentario:

```html
<!--Password: winter-->
```

✅ Resultado:
- Credencial candidata: `winter`

> En el mundo real, esto pasa muchísimo:
> - Comentarios dejados por devs
> - Keys en JS
> - Endpoints ocultos
> - Credenciales en repositorios

---

## 11. `robots.txt` como fuente de pistas

Abrir:

```text
http://192.168.247.128/robots.txt
```

Contenido:

```text
/cgi-bin/root_shell.cgi
/cgi-bin/tracertool.cgi
/cgi-bin/*
```

### ¿Qué es robots.txt?

Es un archivo para “robots” (Google, Bing) indicando qué NO deben indexar.

En seguridad:
- suele filtrar rutas “sensibles”
- paneles internos
- herramientas de diagnóstico
- endpoints no pensados para público

En CTF:
- casi siempre es una pista directa.

---

## 12. CGI `root_shell.cgi` (pista / broma)

Abrir:

```text
http://192.168.247.128/cgi-bin/root_shell.cgi
```

Aparece “UNDER CONSTRUCTION” con comentarios, pero sin funcionalidad explotable (por ahora).

---

## 13. CGI `tracertool.cgi` y Command Injection

Abrir:

```text
http://192.168.247.128/cgi-bin/tracertool.cgi
```

📷 **Imagen 4 — Formulario de tracertool**
![tracertool](images/04-tracertool-form.png)

Este formulario pide una IP para “trace”.

### ¿Qué hace realmente?
Por el comportamiento observado, el servidor ejecuta algo equivalente a:

```bash
traceroute <INPUT_DEL_USUARIO>
```

Si el input no se sanitiza, puedes intentar **inyectar comandos**.

### 13.1 Prueba de concatenación de comandos

Ejemplo:

```
192.168.247.128; whoami; pwd
```

Salida:

- `apache`
- `/var/www/cgi-bin`

✅ Esto confirma:

- El proceso web corre como usuario `apache`
- Estamos en el directorio CGI
- Existe **inyección de comandos** (RCE a nivel de usuario web)

📌 Importante (concepto):
- `;` ejecuta comandos en secuencia (siempre).
- `&&` ejecuta el segundo solo si el primero fue exitoso.
- `||` ejecuta el segundo solo si el primero falla.
- `|` **no** es “concatenación”, es **pipe**: salida de A → entrada de B.

### 13.2 Enumeración del directorio CGI

```
; ls -la
```

Salida:

```text
-rwxr-xr-x root_shell.cgi
-rwxr-xr-x tracertool.cgi
```

### 13.3 Lectura de `/etc/passwd` (enumeración de usuarios)

Intento:

```
; cat /etc/passwd
```

En tu caso aparecía output “alterado” (posible filtrado o formateo del CGI).  
Pero con `tail` sí funcionó:

```
; tail -n 30 /etc/passwd
```

Salida mostró usuarios con shell:

- `RickSanchez` → `/bin/bash`
- `Morty` → `/bin/bash`
- `Summer` → `/bin/bash`

Regla práctica:
- `/bin/bash` = usuario interactivo (posible login)
- `/sbin/nologin` = usuario de servicio (no login)

✅ Esto nos da objetivos para SSH.

---

## 14. Puerto 9090/tcp (Cockpit) y flag visible

Se accede a:

```text
http://192.168.247.128:9090
```

📷 **Imagen 6 — Cockpit mostrando flag**
![Cockpit 9090](images/06-cockpit-9090-flag.png)

Se observa:

```text
FLAG {THERE IS NO ZEUS, IN YOUR FACE!} - 10 POINTS
```

> En CTFs a veces hay flags “regaladas” para confirmar que vas bien.

---

## 15. Enumeración adicional de puertos (nmap rápido)

```bash
nmap 192.168.247.128 -p-
```

Hallazgos:

- 13337/tcp
- 22222/tcp
- 60000/tcp

### 15.1 Enumeración de esos puertos con `-sCV`

```bash
nmap 192.168.247.128 -p13337,22222,60000 -sCV
```

Resultados clave:

- `13337/tcp` devuelve una flag directamente:
  - `FLAG:{TheyFoundMyBackDoorMorty}-10Points`

- `22222/tcp` es **SSH real** (OpenSSH 7.5)

- `60000/tcp` devuelve un banner:
  - `Welcome to Ricks half baked reverse shell...`

Interpretación:

- 22/tcp probablemente es **bait** o servicio “cortado”
- 22222/tcp es el **SSH usable**
- 60000 parece una shell custom o servicio interactivo (lo explotaremos después)

---

## 16. Acceso por SSH al puerto alternativo 22222 (credenciales reutilizadas)

Como teníamos:
- usuarios (de `/etc/passwd`)
- contraseña potencial `winter`

probamos:

```bash
ssh Summer@192.168.247.128 -p 22222
```

La combinación que funcionó fue:

✅ `Summer : winter`

Tras entrar:

```bash
whoami
pwd
ls -la
```

Se encuentra una flag en `/home/Summer/FLAG.txt`:

```text
FLAG{Get off the high road Summer!} - 10 Points
```

> Nota: Esto **no es** reverse shell; esto es una **sesión SSH** interactiva (shell remota “legítima”).

---

## 17. Post-explotación inicial: sudo y escalada

Revisión típica:

```bash
sudo -l
```

Este comando sirve para ver si el usuario puede ejecutar algo como root con `sudo`.

En este caso:

```text
Sorry, user Summer may not run sudo on localhost.
```

➡️ No hay escalada directa por sudo.

---

## 18. Pivot lateral: exploración de otros usuarios (Morty / RickSanchez)

En `/home`:

```bash
cd /home
ls
```

Aparecen:

- `Morty`
- `RickSanchez`
- `Summer`

### 18.1 Exploración del home de RickSanchez

Rutas interesantes:

- `RICKS_SAFE/`
- `ThisDoesntContainAnyFlags/`

En `ThisDoesntContainAnyFlags/NotAFlag.txt` era un “trolleo”, no una flag real.

### 18.2 Binario `safe` en `RICKS_SAFE`

```bash
cd /home/RickSanchez/RICKS_SAFE
file safe
```

`file` sirve para identificar el tipo de archivo (texto, binario, ELF, etc.).

Salida:
- ELF 64-bit ejecutable (binario Linux)

Al intentar visualizarlo directamente, se ve ilegible porque **no es texto plano**.

➡️ Este binario sugiere que habrá que:
- ejecutarlo
- analizar strings del binario
- o hacer reversing básico

Lo retomaremos más adelante.

---

## 19. Artefactos en el home de Morty: ZIP y JPG

En `/home/Morty`:

- `journal.txt.zip`
- `Safe_Password.jpg`

Esto en CTF suele ser:
- credenciales
- pistas
- o material para extraer info (zip con password, estego en imagen, etc.)

---

## 20. Extracción de archivos: SCP vs servidor HTTP (lo que hiciste)

### 20.1 ¿Qué es `scp`?

`scp` (Secure Copy Protocol) copia archivos a través de SSH.

Formato:

```bash
scp -P <PUERTO> <USUARIO>@<IP>:/ruta/remota ./destino_local
```

**Ojo con el detalle:**
- `-P` (mayúscula) es el puerto.
- `-p` (minúscula) preserva tiempos/permisos (no es lo mismo).

### 20.2 Primer intento (desde Morty) y fallo de permisos

Intentaste copiar como `summer` desde el directorio de Morty, pero fallaba (credenciales/permiso/host key flow).  
La copia correcta se hizo usando el usuario Summer:

```bash
scp -P 22222 Summer@192.168.247.128:/home/Morty/journal.txt.zip ./
scp -P 22222 Summer@192.168.247.128:/home/Morty/Safe_Password.jpg ./
```

✅ Archivos copiados al home de Summer (en la máquina víctima).

### 20.3 Transferencia a Kali usando `python3 -m http.server`

Como alternativa rápida (muy típica en CTF), levantaste un servidor HTTP temporal:

Intento en puerto 80:

```bash
python3 -m http.server 80
```

Falla con:

```text
PermissionError: [Errno 13] Permission denied
```

#### ¿Por qué falla?
En Linux, los puertos **0–1023** son **privilegiados**:
- Solo root (o procesos con capacidad especial) pueden abrirlos.

Solución: usar un puerto >1024, por ejemplo 8080:

```bash
python3 -m http.server 8080
```

En Kali descargas con `wget`:

```bash
wget http://192.168.247.128:8080/Safe_Password.jpg
```

---

## 21. Análisis del JPG: `exiftool` y `strings`

### 21.1 Metadatos con `exiftool`

```bash
exiftool Safe_Password.jpg
```

No se encontró nada relevante en EXIF (metadatos típicos de cámara/edición).

### 21.2 “Radiografía” rápida con `strings`

`strings` extrae cadenas ASCII legibles de un binario/archivo.

```bash
strings Safe_Password.jpg
```

Aquí apareció algo crítico:

```text
The Safe Password: File: /home/Morty/journal.txt.zip. Password: Meeseek
```

✅ Esto nos da:
- Archivo protegido: `/home/Morty/journal.txt.zip`
- Password del ZIP: `Meeseek`

> En CTFs esto puede estar embebido como texto en el archivo (a veces por esteganografía simple, a veces por edición intencional).

---

## Estado actual y próximos pasos

Hasta aquí tenemos:

### Flags conseguidas
- FTP: `FLAG{Whoa this is unexpected} - 10 Points`
- Web `/passwords/`: `FLAG{Yeah d- just don't do it.} - 10 Points`
- Cockpit 9090: `FLAG {THERE IS NO ZEUS, IN YOUR FACE!} - 10 POINTS`
- Backdoor 13337: `FLAG:{TheyFoundMyBackDoorMorty}-10Points`
- SSH Summer: `FLAG{Get off the high road Summer!} - 10 Points`

### Credenciales encontradas
- Password desde HTML: `winter`
- ZIP password desde JPG: `Meeseek`
- Usuario válido por SSH: `Summer` (en puerto 22222)

### Vectores pendientes / interesantes
- Servicio `60000/tcp` (reverse shell “half baked”)
- ZIP `journal.txt.zip` (ya tenemos password)
- Binario `safe` (RICKS_SAFE) → probablemente contiene credencial/flag
- Pivot a `Morty` o `RickSanchez` (escalada lateral)
- Escalada a root (privesc) aún pendiente

---

## Siguiente paso que necesito de ti

Pásame lo que hiciste **después** de obtener la contraseña del ZIP (`Meeseek`), por ejemplo:

- comando para descomprimir (`unzip` / `7z`)
- contenido del `journal.txt`
- si encontraste credenciales/keys
- y si tocaste el puerto 60000 o el binario `safe`

Con eso continúo el write-up y lo dejamos impecable.


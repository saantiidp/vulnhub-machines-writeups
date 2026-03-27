# Matrix 3 — Writeup extremadamente detallado

## Descripción original y traducción

**Original**

> Machine Details: Matrix is a medium level boot2root challenge Series of MATRIX Machines. The OVA has been tested on both VMware and Virtual Box.  
> Flags: Your Goal is to get root and read /root/flag.txt  
> Networking: DHCP: Enabled IP Address: Automatically assigned  
> Hint: Follow your intuitions ... and enumerate!

**Traducción**

Matrix es una máquina **boot2root** de dificultad media de la serie MATRIX.  
La OVA ha sido probada en VMware y VirtualBox.  
El objetivo es conseguir acceso **root** y leer el archivo `/root/flag.txt`.  
La red usa **DHCP**, por lo que la IP se asigna automáticamente.  
La pista principal es: **sigue tu intuición y enumera**.

---

## Qué significa boot2root

Una máquina *boot2root* está pensada para practicar el ciclo completo de explotación:

1. identificar la IP de la víctima;
2. enumerar puertos y servicios;
3. encontrar una vía de entrada inicial;
4. conseguir shell;
5. escalar privilegios;
6. terminar como root.

No es una sola vulnerabilidad aislada. Es una cadena de pasos.

---

## Red NAT y por qué la usamos

La Kali y la víctima están en **NAT**.

### Qué es NAT

NAT significa *Network Address Translation*.  
En VMware, esto crea una red privada virtual. Dentro de esa red:

- la Kali y la víctima pueden verse entre sí;
- ambas pueden salir a internet;
- pero no aparecen directamente en tu red física real.

### Por qué es buena idea en labs

- Aísla el laboratorio.
- Reduce ruido.
- Evita tocar equipos reales de tu red.
- Hace más fácil identificar cuál es la víctima.

---

## Ver nuestra IP con `ip a`

Lanzamos:

```bash
ip a
```

### Qué es `ip`

`ip` es una utilidad moderna de Linux para consultar y gestionar red.  
Sustituye muchas veces a herramientas antiguas como `ifconfig`.

### Qué hace `ip a`

La `a` significa *address*.  
Te muestra:

- interfaces;
- direcciones IPv4 e IPv6;
- estado;
- MAC address.

### Resultado útil

En `eth0` vemos:

```bash
inet 192.168.184.128/24
```

Eso significa:

- IP atacante: `192.168.184.128`
- máscara: `/24`
- red: `192.168.184.0/24`

`/24` equivale a `255.255.255.0`.

---

## Descubrir la IP de la víctima con Nmap

Comando:

```bash
sudo nmap -n -sn 192.168.184.128/24
```

### Qué es Nmap

**Nmap** es una herramienta de enumeración de red.  
Sirve para:

- descubrir hosts;
- escanear puertos;
- detectar servicios;
- identificar versiones.

### Explicación de las flags

#### `sudo`
Permite a Nmap usar ciertas funciones de red que requieren privilegios.

#### `-n`
No resuelve DNS.  
Va más rápido y aquí no necesitamos nombres de host.

#### `-sn`
Hace solo descubrimiento de hosts.  
No escanea puertos todavía.

#### `192.168.184.128/24`
Es el rango de red que vamos a revisar.

### Resultado

Aparecen:

- `192.168.184.1`
- `192.168.184.2`
- `192.168.184.133`
- `192.168.184.254`
- `192.168.184.128`

### Por qué la víctima es `192.168.184.133`

- `.128` es nuestra Kali.
- `.1`, `.2` y `.254` suelen ser elementos internos de VMware NAT.
- Por descarte, la víctima es `.133`.

---

## Escaneo completo de puertos

Comando:

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.184.133
```

### Explicación de las flags

- `-p-`: todos los puertos TCP.
- `--open`: solo puertos abiertos.
- `-sC`: scripts por defecto de Nmap.
- `-sV`: detección de versiones.
- `-Pn`: asume que el host está activo.
- `-T5`: modo muy rápido y agresivo.
- `-vvv`: mucha verbosidad.
- `-oN fullscan`: guarda salida en un archivo.

### Resultado

```text
80/tcp   open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
6464/tcp open  ssh     OpenSSH 7.7
7331/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
```

---

## Interpretación inicial de puertos

### Puerto 80
HTTP servido con `SimpleHTTPServer`.

### Qué es `SimpleHTTPServer`

En Python 2, `SimpleHTTPServer` es un servidor web muy simple que sirve archivos estáticos del directorio actual.  
Ejemplo típico:

```bash
python -m SimpleHTTPServer
```

Suele implicar:

- directorios listados;
- estructura simple;
- contenido estático;
- poca lógica backend.

### Puerto 6464
SSH en puerto no estándar.

### Qué es SSH

**SSH** significa *Secure Shell*.  
Sirve para conectarte remotamente a una terminal, administrar sistemas y transferir archivos.

Por defecto va en el 22, pero aquí está en el 6464.

### Puerto 7331
Otro servicio web servido con `SimpleHTTPServer`.  
Probablemente apunta a un directorio diferente.

---

## Puerto 80: revisión de la página

Abrimos:

```text
http://192.168.184.133:80
```

y vemos la **imagen 1**.

![Imagen 1](files/imagen_1_puerto_80.png)

La pista temática vuelve a ser el conejo blanco.

---

## Enumeración con `ffuf`

Comando:

```bash
ffuf -u http://192.168.184.133/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

### Qué es `ffuf`

**ffuf** es una herramienta de fuzzing web.  
Sirve para descubrir:

- directorios;
- archivos;
- recursos ocultos;
- parámetros.

### Explicación de flags

- `-u`: URL objetivo. `FUZZ` se sustituye por palabras del diccionario.
- `-c`: color.
- `-w`: wordlist.
- `-t 100`: 100 hilos concurrentes.

### Resultado

Encuentra:

```text
assets
Matrix
```

---

## Revisión de `/assets`

En:

```text
http://192.168.184.133/assets/
```

vemos:

- `css`
- `fonts`
- `img`
- `js`
- `vendors`

### Qué significa “frontend”

Cuando hablamos de frontend hablamos de archivos del lado del navegador:

- `css`: estilos visuales;
- `fonts`: tipografías;
- `img`: imágenes;
- `js`: JavaScript cliente;
- `vendors`: librerías de terceros.

No parecen datos sensibles de backend, pero pueden contener pistas.

---

## Hallazgo en `/assets/img`

Dentro de `img` aparecen:

- `.gitkeep`
- `Matrix_can-show-you-the-door.png`

La imagen muestra un conejo blanco, como en la **imagen 2**.

![Imagen 2](files/imagen_2_conejo_blanco.png)

### Qué es `.gitkeep`

`.gitkeep` es una convención usada para forzar que Git guarde un directorio vacío.  
No es una vulnerabilidad; solo un archivo de mantenimiento.

### Por qué importa el conejo

La web te decía “Follow the White Rabbit”.  
Encontrar una imagen de un conejo blanco refuerza que la temática es una pista real de enumeración.

---

## Revisión de `/Matrix`

Ruta:

```text
http://192.168.184.133/Matrix/
```

Aparece una estructura rara con letras:

```text
4/ 6/ c/ d/ e/ i/ k/ n/ o/ u/ v/ w/
```

### La idea feliz: Neo

En vez de recorrerlo a ciegas, piensas:

> ¿Quién es el protagonista de Matrix?

**Neo**

Entonces pruebas:

```text
/Matrix/n/e/o/
```

Dentro aparecen directorios del 1 al 9.

Probando, encuentras:

```text
/Matrix/n/e/o/6/4/
```

donde existe:

```text
secret.gz
```

El detalle `64` además encaja con la costumbre del autor de usar ese número.

---

## El archivo `secret.gz` y por qué no era realmente un gzip

Al intentar descomprimirlo, falla.

### Explicación

Una extensión no garantiza el contenido real del archivo.  
Que se llame `.gz` no implica que sea gzip.

### Herramienta correcta: `file`

Comando:

```bash
file secret.gz
```

### Qué hace `file`

`file` identifica el tipo real del archivo usando su contenido interno, no solo la extensión.

### Resultado

```text
secret.gz: ASCII text
```

Eso significa que el archivo es texto plano.

---

## Leer el archivo

Lo renombras a `.txt` y haces:

```bash
cat secret.txt
```

Contenido:

```text
admin:76a2173be6393254e72ffa4d6df1030a
```

Tenemos un usuario y un hash.

---

## Herramienta: hashes.com / Hash Identifier

Has pedido que quede explicado en detalle.

### Qué es `hashes.com`

Es una web con herramientas para trabajar con hashes:

- identificación de algoritmo;
- búsqueda en bases de datos;
- utilidades de cracking.

### Qué hace `Hash Identifier`

Le pegas una cadena y trata de decirte qué algoritmo **podría** producir ese formato.

Se basa en:

- longitud,
- prefijos,
- caracteres,
- estructura.

### Importante
**Identificar** el tipo de hash no es lo mismo que **crackear** el hash.

Aquí nos dice que probablemente es **MD5**.

---

## Qué es MD5

MD5 es una función hash antigua que produce una salida de 128 bits, normalmente representada como 32 caracteres hexadecimales.

Ejemplo:

```text
76a2173be6393254e72ffa4d6df1030a
```

### Diferencia entre hash y cifrado

- Cifrado: se puede revertir con clave.
- Hash: no está pensado para revertirse.

Lo que haces en la práctica es:
- comparar contra una base de datos;
- usar diccionario;
- o fuerza bruta.

En este caso, el valor resuelto es:

```text
passwd
```

Así obtenemos:

- usuario: `admin`
- contraseña: `passwd`

---

## Acceso al puerto 7331 con HTTP Basic Auth

Probamos esas credenciales en:

```text
http://192.168.184.133:7331/
```

y entramos, como en la **imagen 3**.

![Imagen 3](files/imagen_3_panel_7331.png)

### Qué es HTTP Basic Authentication

Es un método simple de autenticación HTTP.

El navegador pide usuario y contraseña, y luego manda una cabecera `Authorization` con las credenciales codificadas en Base64.

### Importante
Base64 no cifra de verdad. Por sí solo no protege nada.

---

## Fuzzing del 7331 sin credenciales

Comando:

```bash
ffuf -u http://192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

Todo devuelve:

```text
401 Unauthorized
```

### Qué significa

No que la ruta no exista, sino que el servidor exige autenticación.

---

## Fuzzing del 7331 con credenciales en la URL

Comando:

```bash
ffuf -u http://admin:passwd@192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

### Por qué funciona

Porque ese formato permite que el cliente construya automáticamente la autenticación Basic.

Esto funciona justo porque el servicio usa **HTTP Basic Authentication**.

### Resultado

Aparecen:

```text
data
assets
```

---

## Descarga del archivo `data`

En:

```text
http://192.168.184.133:7331/data/
```

hay un archivo llamado `data`.

Lo descargas y ejecutas:

```bash
file data
```

Resultado:

```text
PE32 executable for MS Windows 6.00 (GUI), Intel i386 Mono/.Net assembly, 3 sections
```

### Interpretación

- `PE32`: ejecutable Windows.
- `i386`: 32 bits.
- `Mono/.Net assembly`: ensamblado .NET / Mono.

Aunque la víctima sea Linux, esto no está ahí para ejecutarse allí, sino para analizarlo.

---

## Ghidra, explicado en detalle

### Qué es Ghidra

**Ghidra** es una suite de ingeniería inversa desarrollada por la NSA y liberada como open source.

Sirve para analizar binarios compilados cuando no tienes el código fuente.

### Para qué sirve

- malware analysis,
- reverse engineering,
- CTFs,
- entender lógica interna,
- buscar strings y secretos,
- estudiar imports, funciones y clases.

### Qué hace internamente

Cuando importas un binario:

1. detecta arquitectura y formato;
2. lo desensambla;
3. intenta etiquetar funciones y símbolos;
4. extrae strings;
5. si puede, decompila a pseudocódigo parecido a C.

### Por qué aquí era útil

Porque `data` era un ensamblado `.NET`, y ese tipo de binarios suele conservar muchas cadenas y estructura legible.

---

## Crear un proyecto en Ghidra

Al abrir Ghidra ves la **imagen 4**.

![Imagen 4](files/imagen_4_ghidra_sin_proyecto.png)

No hay proyecto activo.

### Pasos

1. `File > New Project`
2. `Non-shared Project`
3. `Next`
4. nombre, por ejemplo `Matrix`
5. `Finish`

Resultado: proyecto creado, como en la **imagen 5**.

![Imagen 5](files/imagen_5_proyecto_creado.png)

---

## Importar el archivo en Ghidra

Haces:

```text
File > Import File
```

Importas `data` y aparece dentro del proyecto, como en la **imagen 6**.

![Imagen 6](files/imagen_6_archivo_importado.png)

Doble clic y se abre la interfaz principal, como en la **imagen 7**.

![Imagen 7](files/imagen_7_ghidra_abierto.png)

---

## Entender la interfaz de Ghidra

### `Listing`
Vista del contenido desensamblado y estructuras.

### `Decompiler`
Pseudocódigo reconstruido.

### `Program Tree`
Secciones del binario.

### `Symbol Tree`
Imports, exports, functions, labels, classes, namespaces.

### Consola
Mensajes del análisis.

La pista importante aquí salió del árbol de símbolos y las cadenas.

---

## Explicación de Labels, Classes, etc.

En la **imagen 8** aparecen zonas como:

- Imports
- Exports
- Functions
- Labels
- Classes
- Namespaces

![Imagen 8](files/imagen_8_panel_symbols_classes.png)

### Imports
Funciones externas usadas por el programa.

### Exports
Funciones que el binario exporta.

### Functions
Funciones detectadas dentro del programa.

### Labels
Etiquetas simbólicas para direcciones o elementos del binario.  
Sirven para que no tengas que leer solo memoria en hexadecimal.

### Classes
Muy importantes en `.NET`, porque pueden reflejar estructura lógica del programa.

### Namespaces
Espacios de nombres del código.

---

## Hallazgo en Ghidra

Revisando cadenas, encuentras:

```text
guest:7R1n17yN30
```

Tiene formato clarísimo de credenciales.

Conclusión:

- usuario: `guest`
- contraseña: `7R1n17yN30`

---

## Acceso por SSH al puerto 6464

Comando:

```bash
ssh guest@192.168.184.133 -p 6464
```

### Explicación de la sintaxis

- `guest@192.168.184.133`: usuario + host
- `-p 6464`: usar ese puerto en vez del 22 por defecto

La conexión funciona, pero caemos en una **rbash**.

---

## Qué es `rbash`

`rbash` significa **restricted bash**.

Es una Bash restringida pensada para limitar al usuario.

Puede impedir:

- ejecutar rutas con `/`,
- cambiar PATH,
- lanzar otros shells,
- usar ciertos comandos.

Por eso, por ejemplo, `whoami` no funciona.

---

## Escape con `vi`

Pruebas con `vi` y dentro ejecutas:

```vim
:!/bin/bash
```

### Por qué funciona

`vi` permite ejecutar comandos externos del sistema con `:!`.

Si lanzas `/bin/bash`, abres una Bash nueva, fuera de la lógica restringida original.

---

## Escape mejor con SSH y `bash --noprofile`

Comando:

```bash
ssh guest@192.168.184.133 -p 6464 -t "bash --noprofile"
```

### Qué hace `-t`
Fuerza un pseudo-terminal interactivo.

### Qué hace `bash --noprofile`
Arranca una Bash limpia sin cargar perfiles del usuario.

### Por qué puede saltarse la rbash

Muchas restricciones dependen del shell de login o de archivos de perfil.  
Si fuerzas otra Bash limpia, evitas esa lógica.

Resultado: una shell mucho más usable.

---

## Enumeración como guest

Revisas `.bash_history`, pero no hay nada importante.

Entonces haces:

```bash
sudo -l
```

Resultado:

```text
(root) NOPASSWD: /usr/lib64/xfce4/session/xfsm-shutdown-helper
(trinity) NOPASSWD: /bin/cp
```

La línea interesante es:

```text
(trinity) NOPASSWD: /bin/cp
```

---

## SSH con claves, explicado muy bien

### Dos piezas

1. **Clave privada**
   La guardas tú. Es secreta.

2. **Clave pública**
   Se copia al servidor dentro de la cuenta remota.

### Archivo importante
En el servidor, SSH consulta:

```bash
~/.ssh/authorized_keys
```

Si la clave pública está ahí, la privada correspondiente puede autenticarse.

### Diferencia con `known_hosts`

#### `authorized_keys`
Claves públicas autorizadas a entrar como ese usuario.

#### `known_hosts`
Huellas de servidores conocidos, guardadas en el cliente.

Para pivotar a otro usuario te interesa `authorized_keys`, no `known_hosts`.

---

## Generar claves con `ssh-keygen`

Comando:

```bash
ssh-keygen
```

### Qué es `ssh-keygen`
Herramienta estándar para generar pares de claves SSH.

### Qué crea

- `id_rsa` → clave privada
- `id_rsa.pub` → clave pública

---

## Abusar de `sudo -u trinity /bin/cp`

Comando conceptual:

```bash
sudo -u trinity /bin/cp id_rsa.pub /home/trinity/.ssh/authorized_keys
```

### Qué hace exactamente

- `sudo -u trinity`: ejecuta el comando como `trinity`
- `/bin/cp`: binario autorizado
- `id_rsa.pub`: tu clave pública
- destino: `authorized_keys` de `trinity`

### Qué logras

Metes tu clave pública dentro de las claves autorizadas de `trinity`.

Eso significa que tu clave privada correspondiente podrá autenticarse como ese usuario.

---

## Conexión como `trinity`

Comando:

```bash
ssh trinity@127.0.0.1 -i /home/guest/.ssh/id_rsa -p 6464
```

### Explicación

- `trinity@127.0.0.1`: login como `trinity` en localhost
- `-i`: usar esa clave privada
- `-p 6464`: puerto SSH correcto

### Por qué funciona

Porque el servidor compara tu clave privada con la pública que ahora está en:

```bash
/home/trinity/.ssh/authorized_keys
```

y encuentra coincidencia.

Resultado: accedes como `trinity`.

---

## Enumeración como `trinity`

Comando:

```bash
sudo -l
```

Resultado:

```text
(root) NOPASSWD: /home/trinity/oracle
```

Eso significa que `trinity` puede ejecutar como root el archivo:

```bash
/home/trinity/oracle
```

---

## El fallo definitivo: `oracle` no existe

Compruebas `/home/trinity` y el archivo no está.

### Por qué eso es explotable

Si `sudo` te autoriza una ruta fija como root y tú puedes crearla, entonces decides qué ejecuta root.

Ese es el bug de diseño.

---

## Crear `oracle` y conseguir root

Comandos:

```bash
echo "/bin/bash" > oracle
chmod 777 oracle
sudo ./oracle
```

### Explicación

#### `echo "/bin/bash" > oracle`
Crea un archivo llamado `oracle` cuyo contenido es `/bin/bash`.

#### `chmod 777 oracle`
Le da permisos totales. Lo importante es que sea ejecutable.

#### `sudo ./oracle`
Ejecuta ese archivo como root.

### Qué pasa realmente

Root ejecuta el archivo.  
El archivo llama a Bash.  
Obtienes una shell root.

---

## Leer la flag final

Ya como root:

```bash
cd /root
cat flag.txt
```

y obtienes la flag.

---

## Resumen de herramientas y utilidad

- `ip`: ver interfaces e IPs.
- `nmap`: descubrir hosts, puertos y servicios.
- `ffuf`: fuzzing web.
- `file`: identificar tipo real de archivo.
- `cat`: leer texto.
- `hashes.com/hash_identifier`: identificar posible algoritmo de hash.
- `hashes.com`: consultar si un hash está resuelto.
- `ssh`: terminal remota segura.
- `ssh-keygen`: generar claves SSH.
- `sudo`: ejecutar como otro usuario.
- `cp`: copiar archivos; aquí permitió el pivoting.
- `chmod`: cambiar permisos.
- `ghidra`: ingeniería inversa del binario `.NET`.
- `vi`: editor que además permite ejecutar comandos externos.

---

## Cadena completa de ataque

1. descubrir IP víctima;
2. enumerar puertos;
3. revisar puerto 80;
4. fuzzear y encontrar `assets` y `Matrix`;
5. seguir la pista de Neo;
6. encontrar `secret.gz`;
7. ver con `file` que era texto;
8. obtener `admin:hash`;
9. identificar y crackear MD5;
10. entrar al 7331 con Basic Auth;
11. fuzzear autenticado;
12. descargar `data`;
13. analizar `data` con Ghidra;
14. obtener `guest:7R1n17yN30`;
15. entrar por SSH;
16. escapar de `rbash`;
17. revisar `sudo -l`;
18. generar claves SSH;
19. copiar la pública al `authorized_keys` de `trinity`;
20. entrar como `trinity`;
21. revisar `sudo -l`;
22. detectar el permiso sobre `oracle`;
23. crear `oracle`;
24. ejecutarlo con `sudo`;
25. obtener root;
26. leer la flag.

---

## Conclusión

Aquí lo importante no era solo “hacer cosas”, sino entender:

- qué herramienta estabas usando;
- qué te estaba devolviendo;
- por qué ese resultado era útil;
- y cómo se conectaba con el siguiente paso.

Eso es lo que hace realmente bueno un writeup formativo.

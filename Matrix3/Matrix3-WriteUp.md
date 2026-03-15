# Matrix 3 (VulnHub) en VMware — WriteUp completo

> **Aviso (ético y legal):** Todo lo descrito aquí está pensado **solo para laboratorio/CTF** y en entornos controlados. No apliques estas técnicas sobre sistemas reales sin autorización.

---

## Descripción de la máquina

**Descripción**  
**Detalles de la máquina:**  
**Matrix** es un desafío **boot2root** de dificultad media, perteneciente a la serie de máquinas **MATRIX**. El archivo **OVA** ha sido probado tanto en **VMware** como en **VirtualBox**.

**Flags:** Tu objetivo es conseguir **root** y leer `/root/flag.txt`

**Red:**
- **DHCP:** habilitado
- **Dirección IP:** asignada automáticamente

**Pista:** Sigue tus intuiciones… ¡y enumera!


---

## 1) Configuración en NAT y preparación del laboratorio

Configuramos Kali y la víctima en **adaptador NAT**, igual que en las máquinas anteriores.

### ¿Por qué NAT?

Porque así:
- ambas VMs quedan dentro de una misma red virtual privada,
- reciben IP automáticamente por DHCP,
- se ven entre sí,
- y además pueden salir a Internet a través del servicio NAT de VMware.

Esto es útil porque:
- te aísla del resto de tu red real,
- te deja un entorno limpio,
- y facilita descubrir la IP de la máquina vulnerable.

---

## 2) Identificar nuestra IP y rango de red

Desde Kali hacemos el procedimiento habitual:

```bash
ip a
```

Esto nos sirve para:
- ver nuestras interfaces de red,
- identificar cuál está activa,
- y saber en qué subred estamos trabajando.

En este caso nos interesa `eth0`, y la IP es:

- `192.168.184.128/24`

### Interpretación

- Kali: `192.168.184.128`
- máscara: `/24`
- red: `192.168.184.0/24`

Eso significa que la víctima debería estar en ese mismo rango.

---

## 3) Descubrir la IP de la víctima

Lanzamos:

```bash
sudo nmap -n -sn 192.168.184.128/24
```

### Explicación detallada de flags

- `sudo`  
  Nmap puede usar ciertos métodos de descubrimiento que requieren privilegios.

- `-n`  
  No resuelve DNS.

  Esto significa:
  - no intenta convertir IP en hostname,
  - el escaneo va más rápido,
  - y evitas tráfico extra de resolución.

- `-sn`  
  Hace un **Ping Scan**.

  Es decir:
  - detecta hosts activos,
  - pero **no** escanea puertos.

- `192.168.184.128/24`  
  Le dices que recorra toda la red `192.168.184.0/24`.

### Resultado observado

```text
192.168.184.1
192.168.184.2
192.168.184.133
192.168.184.254
192.168.184.128
```

### ¿Por qué sabemos que la víctima es `192.168.184.133`?

Porque:
- `192.168.184.128` es Kali
- `.1`, `.2` y `.254` son direcciones típicas de la infraestructura NAT de VMware
- la única IP “nueva” o no identificada es `192.168.184.133`

Por tanto, por descarte razonable, la máquina víctima es:

- **`192.168.184.133`**

---

## 4) Escaneo completo de puertos

Con la IP ya identificada, lanzamos el escaneo completo:

```bash
sudo nmap -p- --open -sCV -Pn -T5 -vvv -oN fullscan 192.168.184.133
```

### Explicación detallada de flags

- `-p-`  
  Escanea todos los puertos TCP, del 1 al 65535.

- `--open`  
  Muestra solo los puertos abiertos.

- `-sC`  
  Ejecuta los scripts NSE por defecto.

- `-sV`  
  Intenta identificar versiones y banners de los servicios.

- `-Pn`  
  Asume que el host está activo y no depende del descubrimiento por ping.

- `-T5`  
  Timing agresivo, válido en laboratorio y CTF.

- `-vvv`  
  Muy verboso, para ver con más detalle lo que se va descubriendo.

- `-oN fullscan`  
  Guarda el resultado en un archivo llamado `fullscan`.

### Puertos encontrados

```text
80/tcp   open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
6464/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
7331/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
```

---

## 5) Interpretación de los servicios expuestos

### Puerto 80 — HTTP con `SimpleHTTPServer`
El puerto 80 es el estándar de HTTP.

Pero aquí el servicio no es Apache ni nginx, sino:

- `SimpleHTTPServer 0.6 (Python 2.7.14)`

### ¿Qué es `SimpleHTTPServer`?

Es el servidor web muy simple que viene en Python 2.  
Se suele lanzar con:

```bash
python -m SimpleHTTPServer
```

Eso normalmente:
- expone una carpeta,
- permite servir archivos estáticos,
- y muchas veces muestra **directory listing**.

En un CTF, cuando ves esto, piensas enseguida:
- puede haber rutas con listados,
- archivos directamente descargables,
- y mucha lógica “simple” del lado web.

### Puerto 6464 — SSH
El servicio es:

- `OpenSSH 7.7`

SSH sirve para:
- acceso remoto a terminal,
- administración del sistema,
- sesiones seguras cifradas.

Lo raro aquí no es el servicio, sino el puerto:
- normalmente sería el 22
- aquí está en `6464`

Eso suele significar:
- ocultación básica,
- decisión del autor del laboratorio,
- o simplemente una pista de que no todo estará en sitios estándar.

### Puerto 7331 — otro HTTP con `SimpleHTTPServer`
Otro servidor Python.
Eso sugiere algo importante:
- probablemente **cada puerto sirve carpetas diferentes**
- así que el contenido del 80 y el 7331 puede ser distinto aunque el software sea el mismo.

---

## 6) Web principal en el puerto 80

Abrimos en navegador:

- `http://192.168.184.133:80`

📷 **Imagen 1 — Web principal en 80**
![Main web 80](images/01-main-web-80.png)

Visualmente se parece muchísimo al estilo Matrix1:
- fondo oscuro
- frase “Follow the White Rabbit”
- estética de reto guiado por pistas

---

## 7) Fuzzing sobre el puerto 80

Hacemos un `ffuf` sobre el puerto 80:

```bash
ffuf -u http://192.168.184.133/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

### Explicación detallada de flags

- `-u http://192.168.184.133/FUZZ`  
  URL objetivo. `FUZZ` es el marcador que ffuf sustituye con cada palabra del diccionario.

- `-c`  
  Salida coloreada.

- `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`  
  Wordlist de rutas/directorios.

- `-t 100`  
  Número de hilos concurrentes. Rápido, pero ruidoso.

### Hallazgos

Aparece:

```text
assets  [Status: 301]
Matrix  [Status: 301]
```

Eso ya da dos rutas interesantes.

---

## 8) Revisar `/assets/`

Entramos a:

- `http://192.168.184.133/assets/`

Y vemos un **directory listing**:

```text
css/
fonts/
img/
js/
vendors/
```

### Qué significa esto y por qué lo llamamos “cosas de frontend”

Esas carpetas son típicas de los recursos estáticos de una web:
- `css/` → hojas de estilo
- `fonts/` → tipografías
- `img/` → imágenes
- `js/` → JavaScript
- `vendors/` → librerías de terceros

A simple vista no parece haber lógica de backend ni ficheros sensibles ahí.  
Pero en laboratorios, incluso dentro de frontend, puede haber pistas escondidas:
- nombres de archivos,
- imágenes relevantes,
- comentarios,
- rutas que apuntan a otros servicios.

---

## 9) La carpeta `/img/` y la pista del conejo blanco

Dentro de `/assets/img/` aparecen cosas curiosas:

```text
.gitkeep
Matrix_can-show-you-the-door.png
```

### `.gitkeep`
No es una pista potente por sí sola.  
Suele ser un fichero vacío que se usa para obligar a Git a versionar carpetas vacías.

### `Matrix_can-show-you-the-door.png`
Ese nombre sí es interesante.  
Y cuando lo abres, contiene un **conejo blanco**.

📷 **Imagen 2 — Imagen del conejo blanco**
![White rabbit image](images/02-white-rabbit-image.png)

### Por qué importa tanto

Porque encaja con el mensaje de la portada:

- “Follow the White Rabbit”

Y además el nombre dice:

- `Matrix_can-show-you-the-door`

Es decir:
- Matrix puede mostrarte la puerta,
- pero tú tienes que seguir el camino.

Aquí no hay una explotación todavía, pero sí una pista narrativa muy clara.

---

## 10) La ruta `/Matrix/`

El fuzzing encontró además:

```text
Matrix [Status: 301]
```

Entramos a:

- `http://192.168.184.133/Matrix/`

Y vemos:

```text
4/
6/
c/
d/
e/
i/
k/
n/
o/
u/
v/
w/
```

### Qué vemos aquí

Es una estructura muy rara:
- letras sueltas,
- números sueltos,
- muchas posibles rutas,
- y ninguna explicación evidente.

Enumerarlo a mano sin criterio sería bastante pesado.

---

## 11) La idea feliz: seguir el nombre del protagonista

Aquí entra una idea muy buena y muy de esta serie de máquinas.

Piensas:
- ¿quién es el protagonista de Matrix?
- **Neo**

Así que pruebas la ruta:

- `/Matrix/n/e/o/`

Y dentro aparecen:

```text
1/
2/
3/
4/
5/
6/
7/
8/
9/
```

### Qué significa esto

Ya hay una lógica más clara:
- primero las letras de `neo`
- luego posibles números

Dices que revisaste una a una y que la combinación interesante fue:

- `/Matrix/n/e/o/6/4/`

Dentro encuentras:

```text
secret.gz
```

---

## 12) Descargar `secret.gz` y por qué falla al descomprimir

Descargas:

- `secret.gz`

Y al intentar descomprimirlo te sale:

```text
gzip: /home/kali/Downloads/secret.gz: not in gzip format
```

### ¿Por qué pasa esto?

Porque el archivo tiene extensión `.gz`, pero en realidad **no es un gzip real**.

Es decir:
- el nombre intenta engañarte,
- pero el contenido no corresponde al formato comprimido gzip.

Esto es algo común en CTF:
- cambiar extensiones
- renombrar archivos
- hacerte perder tiempo si te fías solo del nombre

Lo correcto aquí es no asumir nada y usar `file`.

Además, aquí el número `64` aparece otra vez.  
Y encaja también con el nombre del creador / serie que vienes comentando:
- `unknowndevice64`

Eso refuerza que el `6/4` no era una ruta aleatoria.

---

## 13) Mover el archivo y usar `file`

Mueves el archivo a tu directorio de trabajo:

```bash
mv ~/Downloads/secret.gz .
```

Y haces:

```bash
file secret.gz
```

Salida:

```text
secret.gz: ASCII text
```

### Qué significa esto

Aunque se llame `secret.gz`, **no es un comprimido**, sino:
- texto ASCII plano

Por eso `gzip` falla:
- no es que esté corrupto,
- es que nunca fue un gzip de verdad.

Así que lo renombramos a algo coherente:

```bash
mv secret.gz secret.txt
```

Y ahora sí lo leemos:

```bash
cat secret.txt
```

Contenido:

```text
admin:76a2173be6393254e72ffa4d6df1030a
```

---

## 14) Identificar y crackear el hash

Tenemos:
- usuario: `admin`
- un hash: `76a2173be6393254e72ffa4d6df1030a`

No sabemos todavía qué tipo de hash es.

Usas una web de identificación y te dice:
- posible algoritmo: **MD5**

Después usas la parte de crack/decrypt y obtienes:

```text
76a2173be6393254e72ffa4d6df1030a : passwd
```

### Qué significa esto

La credencial en texto claro queda así:

- usuario: `admin`
- contraseña: `passwd`

---

## 15) Probar credenciales en el puerto 7331

Nos vamos al otro HTTP abierto:

- `http://192.168.184.133:7331/`

Y usamos:
- `admin`
- `passwd`

Entramos correctamente.

📷 **Imagen 3 — Acceso en el puerto 7331**
![Login 7331](images/03-login-7331.png)

---

## 16) Fuzzing en el puerto 7331 y por qué fallaba con 401

Primero probaste fuzzing normal:

```bash
ffuf -u http://192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

Pero todas las rutas devolvían:

- `401 Unauthorized`

### ¿Por qué?

Porque el contenido está protegido por **HTTP Basic Authentication**.

### Qué es HTTP Basic Authentication

Es un método muy simple de autenticación del protocolo HTTP:
- el navegador te pide usuario y contraseña
- esas credenciales se envían en la cabecera `Authorization`
- si son correctas, el servidor deja pasar

Mientras `ffuf` no tenga esas credenciales:
- cualquier ruta probada seguirá devolviendo 401

---

## 17) Fuzzing en el 7331 con credenciales embebidas

Entonces ajustas el comando:

```bash
ffuf -u http://admin:passwd@192.168.184.133:7331/FUZZ -c -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 100
```

### Explicación de por qué funciona

La forma:

```text
http://usuario:password@host:puerto/
```

permite que el cliente HTTP use esas credenciales para la autenticación básica.

En este caso funciona porque el servicio está usando **HTTP Basic Auth**.  
Si fuese otro sistema de login (formulario, tokens, cookies, etc.), este truco no serviría.

### Resultado

Aparecen:

```text
data    [Status: 301]
assets  [Status: 301]
```

La ruta que más llama la atención es:

- `data`

---

## 18) Revisar `/data/`

Accedes a:

- `http://192.168.184.133:7331/data/`

Y ves:

```text
Directory listing for /data/

data
```

Cuando haces clic en `data`, se descarga automáticamente un fichero llamado:

- `data`

Lo mueves a tu carpeta de trabajo:

```bash
mv ~/Downloads/data .
```

Y lo analizas con `file`:

```bash
file data
```

Salida:

```text
data: PE32 executable for MS Windows 6.00 (GUI), Intel i386 Mono/.Net assembly, 3 sections
```

---

## 19) ¿Qué significa ese resultado de `file`?

Muchísimas cosas.

### `PE32 executable`
Eso significa que es un ejecutable tipo Windows:
- formato PE (Portable Executable)
- equivalente a `.exe`

### `for MS Windows 6.00 (GUI)`
Fue compilado como binario Windows, probablemente pensado para entorno gráfico.

### `Intel i386`
Arquitectura de 32 bits x86.

### `Mono/.Net assembly`
Esto es lo importante:
- no es solo un `.exe` cualquiera
- es un ensamblado `.NET` / Mono

Eso significa que puede ser:
- una aplicación escrita en C#
- ejecutable en entornos .NET o Mono
- y suele ser muy agradecido para reversing

### El problema aparente

La víctima es Linux y el binario es Windows/.NET.  
Eso te desconcierta, porque:
- no puedes ejecutarlo “normalmente” en Kali como si fuera un ELF
- y además ni siquiera parece un binario nativo del sistema víctima

Así que toca **ingeniería inversa**.

---

## 20) Ghidra: qué es y por qué la usamos

Usamos **Ghidra**.

### Qué es Ghidra

Ghidra es una herramienta de **ingeniería inversa** desarrollada por la NSA y liberada como open source.

Sirve para analizar binarios compilados cuando no tienes el código fuente.

### Para qué se usa

- reversing de malware
- análisis de ejecutables
- estudio de funciones internas
- extracción de strings y lógica
- reversing en CTF y máquinas tipo VulnHub/HTB

### Qué puede abrir

- ELF (Linux)
- PE (Windows)
- DLL
- firmware
- .NET/Mono
- y muchas arquitecturas distintas

En este caso nos interesa porque:
- tenemos un ejecutable raro
- parece contener lógica o credenciales
- y Ghidra puede ayudarnos a extraer información sin ejecutarlo

---

## 21) Abrir Ghidra y crear un proyecto

Ejecutas:

```bash
ghidra
```

Y te sale la ventana inicial sin proyecto.

📷 **Imagen 4 — Ghidra sin proyecto activo**
![Ghidra no project](images/04-ghidra-no-project.png)

### Pasos que sigues

1. `File > New Project`
2. `Non-shared Project`
3. `Next`
4. Le pones un nombre, por ejemplo:
   - `Matrix`
5. `Finish`

Después aparece el proyecto creado.

📷 **Imagen 5 — Proyecto creado**
![Ghidra project created](images/05-ghidra-project-created.png)

---

## 22) Importar el archivo `data`

Seleccionas la carpeta del proyecto y haces:

- `File > Import File`
- importas el archivo `data`
- aceptas el análisis

Ahora el binario aparece dentro del proyecto.

📷 **Imagen 6 — Fichero importado**
![Ghidra file imported](images/06-ghidra-file-imported.png)

Le haces doble clic y se abre el **CodeBrowser** de Ghidra.

📷 **Imagen 7 — CodeBrowser**
![Ghidra codebrowser](images/07-ghidra-codebrowser.png)

---

## 23) Qué estamos viendo en Ghidra

En esa interfaz tienes varias zonas:

- árbol del programa
- símbolos
- imports/exports
- funciones
- labels
- decompiler
- listing en ensamblador/desensamblado

Como el objetivo aquí no es rehacer el programa entero, sino encontrar datos útiles, vamos a inspeccionar las partes con más probabilidad de contener:
- strings
- nombres de clases
- recursos
- texto incrustado

---

## 24) Symbol Tree, Labels y por qué importan

A la izquierda aparece un árbol con:
- Imports
- Exports
- Functions
- Labels
- Classes
- Namespaces

📷 **Imagen 8 — Árbol de símbolos**
![Ghidra symbol tree](images/08-ghidra-symbol-tree.png)

### Qué son los `Labels`

En ensamblador y reversing, los **labels** son etiquetas que apuntan a:
- direcciones de memoria
- ubicaciones de código
- bloques de datos
- referencias internas

Sirven para que el análisis sea más legible.  
En vez de trabajar solo con direcciones crudas como:
- `00401000`
- `0040129A`

Ghidra intenta poner nombres o etiquetas útiles para navegar mejor.

### Por qué mirar labels aquí

Porque en binarios .NET/Mono, muchas veces:
- strings,
- recursos,
- nombres internos
quedan reflejados en zonas fáciles de identificar.

Y revisando uno por uno esos labels, aparece algo muy valioso.

---

## 25) Hallazgo importante en Ghidra

Dentro de uno de esos elementos encuentras strings como:

- `Microsoft Sans Serif`
- `Welcome In Matrix V3`

Y, lo más importante:

- `guest:7R1n17yN30`

### Interpretación

Eso tiene toda la pinta de ser:
- usuario: `guest`
- contraseña: `7R1n17yN30`

Esto ya no parece una pista, sino una credencial clara.

---

## 26) Probar las credenciales por SSH

Recordando el nmap, hay un servicio SSH en el puerto `6464`.

Probamos:

```bash
ssh guest@192.168.184.133 -p 6464
```

### Explicación de flags

- `ssh`  
  Cliente SSH.

- `guest@192.168.184.133`  
  Te conectas al host `192.168.184.133` como usuario `guest`.

- `-p 6464`  
  Especifica el puerto remoto.

  Esto es **muy importante**, porque si no lo pones:
  - SSH intentará conectar al puerto por defecto, el `22`
  - y aquí el servicio está en `6464`

### Resultado

Consigues acceder:

```text
guest@matrix:~$
```

---

## 27) La `rbash`: qué es y por qué molesta

Al intentar:

```bash
whoami
```

te sale:

```text
-rbash: whoami: command not found
```

Eso indica que has entrado en una:

- **restricted bash**
- o **rbash**

### Qué es `rbash`

Es una Bash restringida diseñada para limitar lo que el usuario puede hacer.

Suele bloquear o dificultar cosas como:
- cambiar directorio
- modificar `PATH`
- llamar binarios con `/`
- lanzar otras shells
- usar ciertos comandos

No es una protección perfecta, y por eso en CTF se suele intentar escapar de ella.

---

## 28) Escape con `vi` y por qué funciona

Pruebas el mismo truco que te funcionó en Matrix2:
- abrir `vi`
- y dentro ejecutar:

```vim
:!/bin/bash
```

### Por qué funciona

`vi` permite ejecutar comandos del sistema desde dentro del editor.
La secuencia:
- `:` abre modo comando
- `!` ejecuta un comando externo
- `/bin/bash` lanza una bash normal

Si el binario `vi` está permitido y accesible, muchas veces esto rompe el entorno restringido.

### Resultado

Ahora sí te deja usar comandos normales como:

```bash
ls
```

Y ves:

```text
Desktop/  Documents/  Downloads/  Music/  Pictures/  Public/  Videos/  prog/
```

---

## 29) Otra forma mejor: `ssh -t "bash --noprofile"`

También descubres otro enfoque muy bueno:

```bash
ssh guest@192.168.184.133 -p 6464 -t "bash --noprofile"
```

### Explicación detallada

- `ssh guest@192.168.184.133`  
  conexión como `guest`

- `-p 6464`  
  puerto SSH correcto

- `-t`  
  fuerza asignación de pseudo-terminal (PTY)

- `"bash --noprofile"`  
  en vez de entrar al shell por defecto del usuario, le dices a SSH que ejecute directamente ese comando remoto

### ¿Por qué funciona?

Porque muchas restricted bash:
- dependen de cómo se inicializa el shell
- o de archivos de configuración como:
  - `/etc/profile`
  - `~/.bash_profile`
  - `~/.bashrc`

Cuando ejecutas:

```bash
bash --noprofile
```

le estás diciendo a Bash:
- no cargues los archivos de configuración del perfil

Si las restricciones estaban metidas ahí, te las saltas.

### Resultado

Obtienes una bash limpia y sin restricciones:

```text
guest@matrix:~$ whoami
guest
```

Esta forma es muy elegante porque ni siquiera necesitas el truco de `vi`.

---

## 30) Enumeración local como `guest`

Haces el típico:

```bash
ls -la
```

Ves `.bash_history` y lo revisas, pero no tiene nada útil:
- solo comandos tuyos

Luego haces lo más importante:

```bash
sudo -l
```

Salida:

```text
User guest may run the following commands on matrix:
    (root) NOPASSWD: /usr/lib64/xfce4/session/xfsm-shutdown-helper
    (trinity) NOPASSWD: /bin/cp
```

### Qué nos interesa aquí

Lo más útil no es lo de root directamente, sino esto:

```text
(trinity) NOPASSWD: /bin/cp
```

Eso significa:
- como `guest`
- puedes ejecutar **`/bin/cp`**
- haciéndote pasar por el usuario `trinity`
- y sin contraseña

---

## 31) Confirmar que existe `trinity`

Compruebas los homes:

```bash
ls /home
```

Salida:

```text
guest/  trinity/
```

Así que sí:
- `trinity` existe como usuario real del sistema

---

## 32) Qué tiene que ver esto con SSH y las claves

Aquí entra una idea muy buena y muy típica de escalada/movimiento lateral.

### Cómo funciona la autenticación por clave SSH

Cuando haces una conexión SSH por clave:

1. Tu cliente SSH usa **tu clave privada**
2. El servidor revisa el archivo:
   - `~/.ssh/authorized_keys`
   del usuario remoto
3. Si la clave pública correspondiente está ahí, te deja entrar

### Qué archivo importa aquí

En el usuario remoto, el archivo importante es:

- `~/.ssh/authorized_keys`

No `known_hosts`.

### Diferencia importante

- `known_hosts`  
  Guarda huellas de hosts remotos conocidos.
  Sirve para verificar servidores.

- `authorized_keys`  
  Guarda las **claves públicas permitidas para entrar como ese usuario**.

Esto es lo que nos interesa.

---

## 33) Generar nuestras claves SSH como `guest`

Dentro de la cuenta `guest`, vas a `~/.ssh/` y ejecutas:

```bash
ssh-keygen
```

### Qué hace `ssh-keygen`

Genera un par de claves SSH:
- una **clave privada**
- una **clave pública**

En tu caso:
- `id_rsa` → clave privada
- `id_rsa.pub` → clave pública

### Cómo funciona el par de claves

- La **privada** te la quedas tú y nunca deberías compartirla.
- La **pública** puede copiarse al sistema remoto, en `authorized_keys`.

Cuando el servidor ve tu clave pública en `authorized_keys` y tú presentas la privada correcta, te autentica.

Después de generarlas, ves:

```text
id_rsa
id_rsa.pub
```

---

## 34) Preparar la clave pública y copiarla hacia `trinity`

Copias la clave pública a `/tmp` y luego usas el privilegio de sudo permitido:

```bash
sudo -u trinity /bin/cp id_rsa.pub /home/trinity/.ssh/authorized_keys
```

### Explicación detallada

- `sudo -u trinity`  
  Le dices a sudo: ejecuta el siguiente comando como usuario `trinity`.

- `/bin/cp`  
  Es el único binario que te dejan ejecutar como `trinity`.

- `id_rsa.pub`  
  Es tu clave pública, creada como `guest`.

- `/home/trinity/.ssh/authorized_keys`  
  Es el archivo donde SSH busca las claves públicas autorizadas para entrar como `trinity`.

### ¿Qué estás haciendo realmente?

Estás aprovechando que puedes usar `cp` como `trinity` para:
- escribir tu propia clave pública
- dentro del `authorized_keys` de `trinity`

Y eso significa que, a partir de ese momento, tu clave privada de `guest` podrá autenticarse como `trinity`.

---

## 35) Por qué ahora puedes entrar como `trinity`

Una vez que tu clave pública está en:

- `/home/trinity/.ssh/authorized_keys`

puedes usar tu clave privada correspondiente para entrar como `trinity`.

Lo haces contra localhost:

```bash
ssh trinity@127.0.0.1 -i /home/guest/.ssh/id_rsa -p 6464
```

### Explicación de flags

- `ssh`  
  cliente SSH

- `trinity@127.0.0.1`  
  te conectas al host local (`127.0.0.1`) como usuario `trinity`

- `-i /home/guest/.ssh/id_rsa`  
  le indicas qué clave privada usar para autenticación

- `-p 6464`  
  el SSH del sistema escucha en el puerto 6464, no en el 22

### ¿Por qué contra `127.0.0.1`?

Porque ya estás dentro de la máquina.
No hace falta salir y volver a entrar desde Kali:
- puedes reutilizar el propio servicio SSH local
- y hacer un movimiento lateral desde `guest` hacia `trinity`

### ¿Por qué funciona?

Porque:
- tu clave privada `id_rsa`
- corresponde a la clave pública que metiste en `authorized_keys` de `trinity`

Y SSH valida precisamente eso.

### Resultado

Obtienes shell como:

```text
trinity@matrix:~$
```

---

## 36) Sudo como `trinity`

Ahora toca el paso automático de siempre:

```bash
sudo -l
```

Y devuelve:

```text
User trinity may run the following commands on matrix:
    (root) NOPASSWD: /home/trinity/oracle
```

### Qué significa

El usuario `trinity` puede ejecutar como root, sin contraseña:

- `/home/trinity/oracle`

Es decir:
- si existe ese archivo,
- y lo ejecutas con sudo,
- correrá como root

---

## 37) Comprobar si `oracle` existe

Estás en:

```bash
pwd
/home/trinity
```

Pero al listar no aparece el binario `oracle`.

Eso quiere decir algo muy importante:

- el sudoers permite ejecutar `/home/trinity/oracle` como root
- pero no exige que ese archivo ya exista de antemano

Si tú puedes crearlo, puedes controlar **qué se ejecuta como root**.

Y aquí está la clave de la escalada.

---

## 38) Crear `oracle` nosotros mismos

Haces:

```bash
echo "/bin/bash" > oracle
chmod 777 oracle
sudo ./oracle
```

### Qué hace cada comando

#### `echo "/bin/bash" > oracle`
Crea un archivo llamado `oracle` y mete dentro:

```text
/bin/bash
```

Es decir, el archivo contiene una línea que invoca Bash.

#### `chmod 777 oracle`
Le das permisos totales:
- lectura
- escritura
- ejecución
para todos

Esto hace que el archivo sea ejecutable.

#### `sudo ./oracle`
Como `sudo -l` permitía ejecutar `/home/trinity/oracle` como root, al lanzar ese archivo:
- su contenido se ejecuta con privilegios de root

Y como el contenido es `/bin/bash`, acabas lanzando una bash como root.

### Resultado

```text
root@matrix:/home/trinity#
```

✅ Ya eres root.

---

## 39) Leer la flag final

Una vez como root, ya solo queda:

```bash
cd /root
cat flag.txt
```

Y aparece la flag final con ASCII art.

Con esto queda comprometida la máquina completamente.

---

## Conclusiones finales

Esta máquina tiene varias cosas muy interesantes.

### 39.1 La parte web vuelve a premiar la intuición
No fue simplemente fuzzear y encontrar una ruta mágica.
Hubo que:
- seguir la pista del conejo blanco,
- interpretar `/Matrix/`,
- pensar en `neo`,
- y luego recorrer la combinación correcta.

### 39.2 Las extensiones engañan
`secret.gz` no era un gzip real.
Eso recuerda la importancia de no confiar solo en el nombre del archivo y usar siempre `file`.

### 39.3 Reversing ligero con Ghidra
Aquí la parte curiosa fue encontrar un ejecutable `.NET` en una máquina Linux.
En vez de intentar ejecutarlo, el camino correcto fue:
- analizarlo con Ghidra,
- inspeccionar labels/símbolos,
- y extraer credenciales incrustadas.

### 39.4 Escape de `rbash`
Otra vez aparece una bash restringida.
Aquí has mostrado dos maneras útiles de romperla:
- abuso de `vi`
- `ssh -t "bash --noprofile"`

La segunda es especialmente elegante.

### 39.5 Movimiento lateral usando claves SSH
Esta es probablemente la parte más interesante de la privesc:
- no escalas directo a root,
- primero pasas de `guest` a `trinity`
- abusando de permisos sobre `cp`
- y del funcionamiento interno de `authorized_keys`

Es una técnica muy realista y muy útil de entender.

### 39.6 Escalada final por ruta controlable en sudo
El sudoers permitía ejecutar `/home/trinity/oracle` como root.
Como ese archivo no existía y tú podías crearlo, el privilegio era totalmente explotable.

Eso es un fallo clásico de diseño:
- permitir un binario por ruta,
- pero en una ubicación controlable por el usuario.

---

## Resumen técnico de la ruta de explotación

1. Configuración en NAT
2. `ip a` para ver la IP de Kali
3. `nmap -sn` para descubrir la víctima
4. Escaneo completo con Nmap
5. Revisión del puerto 80
6. `ffuf` encuentra `assets` y `Matrix`
7. Pista del conejo blanco
8. Ruta `/Matrix/n/e/o/6/4/`
9. Descarga de `secret.gz`
10. `file` revela que es texto ASCII
11. Obtención de `admin:md5`
12. Identificación/crackeo del hash → `passwd`
13. Login en el puerto 7331
14. `ffuf` con credenciales embebidas en URL
15. Descarga del binario `data`
16. Análisis con Ghidra
17. Extracción de credenciales `guest:7R1n17yN30`
18. SSH a `guest` por el puerto 6464
19. Escape de `rbash`
20. `sudo -l`
21. Descubrimiento de privilegio sobre `/bin/cp` como `trinity`
22. Generación de claves SSH con `ssh-keygen`
23. Copia de la clave pública a `/home/trinity/.ssh/authorized_keys`
24. SSH local como `trinity`
25. `sudo -l` como `trinity`
26. Descubrimiento de que puede ejecutar `/home/trinity/oracle` como root
27. Crear `oracle` con contenido `/bin/bash`
28. Ejecutarlo con sudo
29. Root
30. Lectura de `/root/flag.txt`

---

## Estado del laboratorio

✅ Máquina completada  
✅ Root conseguido  
✅ Flag final leída  
✅ Ruta documentada paso a paso

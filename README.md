# Taller-Linux
Trabajo del Taller Linux

# Despliegue Automatizado de la Intranet con Ansible
Este repositorio contiene la solución completa de automatización para el obligatorio de **Taller de Servidores Linux** (Agosto 2026). El objetivo del proyecto es automatizar de forma integral y robusta la instalación, configuración y despliegue de una aplicación web intranet en PHP (que extrae una lista de cumpleaños) y su respectiva base de datos relacional MariaDB [123, 124, 604].

El diseño arquitectónico de la solución implementa un esquema distribuido de dos servidores gestionados a través de un nodo de control (Ansible Controller) [124, 247, 604]:
- Servidor de Aplicación (CentOS Stream 9)**: IP por DHCP (típicamente `10.0.2.15`), corriendo el servidor web Apache (`httpd`), el procesador `php-fpm` y la extensión de conexión a MySQL [124, 126, 473].
- Servidor de Base de Datos (Ubuntu Server 24.04)**: IP estática `10.0.2.100`, corriendo el motor de base de datos MariaDB [124, 212, 604].

---

## Estructura del Proyecto

La organización del repositorio sigue estrictamente las mejores prácticas para estructurar proyectos de Ansible [128, 606, 634]:

├── collections/
│   └── requirements.yml   # Lista de colecciones de Ansible externas requeridas
├── files/
│   └── cumpleanos.sql     # Script SQL (dump) para inicializar la base de datos
├── inventory/
│   └── hosts.ini          # Archivo de inventario con grupos de servidores y variables
├── playbooks/
│   └── despliegue.yml     # Playbook principal de automatización (despliegue de aplicación y base de datos)
├── templates/
│   └── cumple.j2          # Plantilla Jinja2 de la aplicación PHP (cumple.php) con credenciales dinámicas
├── vars/
│   └── database.yml       # Archivo encriptado con Ansible Vault para proteger credenciales sensibles
└── README.md              # Documentación de instalación, ejecución y validación


---

## Requisitos Previos

Antes de ejecutar la automatización de Ansible, el entorno debe cumplir con las siguientes condiciones [133, 574]:

1. Configuración de Red**:
   - El nodo CentOS Stream 9 (Servidor Web) debe tener asignada una IP en la red del taller (ej. `10.0.2.15`)
   - El nodo Ubuntu Server 24.04 (Base de Datos) debe estar configurado de forma estática con la IP `10.0.2.100`
2. Acceso SSH sin contraseña (Claves Públicas)**:
   - Las claves públicas del usuario administrador del nodo controlador de Ansible deben haberse copiado a ambos servidores gestionados bajo el usuario `sysadmin`:
    
     ssh-copy-id sysadmin@10.0.2.15
     ssh-copy-id sysadmin@10.0.2.100
    
3. Escalamiento de Privilegios (Sudo):
   - El usuario `sysadmin` en ambos nodos gestionados debe tener privilegios de sudo configurados (idealmente con `NOPASSWD` para una ejecución totalmente desatendida).

---

## Instalación de Colecciones

El proyecto requiere colecciones externas que no forman parte de Ansible Core [938, 947, 1370]. Las colecciones necesarias están especificadas en `collections/requirements.yml`:

---
collections:
  - name: ansible.posix
  - name: community.general
  - name: community.mysql


Para instalarlas de forma masiva en el nodo controlador, ejecute:

ansible-galaxy collection install -r collections/requirements.yml


---

## Configuración de Variables Protegidas (Ansible Vault)

De acuerdo a las directivas del obligatorio, no se deben almacenar contraseñas ni datos sensibles en texto plano en el repositorio Git [129, 606]. Para ello, se utiliza **Ansible Vault** para crear un archivo encriptado en `vars/database.yml`:

1. Creación del Vault:

   ansible-vault create vars/database.yml
  
2. Estructura y valores definidos:
   El archivo contiene exactamente las siguientes variables:
   
   DB_SERVER: 10.0.2.100      # IP del servidor Ubuntu donde se aloja MariaDB
   DB_USER: intranet          # Usuario específico de la base de datos para la aplicación
   DB_PASS: intr4n3t          # Contraseña segura del usuario intranet
   DB_DBASE: cumples          # Nombre físico de la base de datos de cumpleaños
   DB_ROOT_PW: aslxlab        # Contraseña segura root   
---
Para evitar almacenar credenciales fijas (hardcoded) en el código de la aplicación, el archivo PHP original se adaptó a la plantilla dinámica `templates/cumple.j2`. La cadena de conexión de MySQL se parametriza usando la sintaxis de Jinja2:
---

## Instrucciones de Ejecución

Una vez cumplidos los prerrequisitos, puede desplegar la infraestructura completa lanzando el playbook principal desde la raíz de su repositorio:

ansible-playbook -i inventory/hosts.ini playbooks/despliegue.yml --ask-become-pass --ask-vault-pass

---

## Resumen de Tareas del Playbook (`playbooks/despliegue.yml`)

El playbook principal realiza las siguientes configuraciones de manera automatizada:

### Servidor de Aplicación (CentOS Stream 9)
1. Instalación de paquetes**: Apache (`httpd`), PHP, procesador FastCGI (`php-fpm`), la extensión `php-mysqlnd` y la librería `python3-libsemanage`.
2. Gestión de Servicios con Systemd**: Habilitar en el arranque e iniciar `httpd` y `php-fpm` de forma persistente.
3. Seguridad de SELinux**: Activar de forma persistente los booleanos de SELinux para autorizar a Apache a realizar conexiones de red hacia la base de datos externa:
   - `httpd_can_network_connect_db`
   - `httpd_can_network_connect`
4. Firewall (Firewalld): Permitir el puerto HTTP (80) de forma permanente e inmediata.
5. Procesamiento de Template: Tomar `templates/cumple.j2`, resolver las variables del Vault e instalarlo en `/var/www/html/index.php` con dueño `apache:apache` y permisos `0644`.

## Servidor de Base de Datos (Ubuntu Server)
1. Instalación de paquetes: Servidor MariaDB (`mariadb-server`) y dependencias Python necesarias para MySQL.
2. Configuración de Red de MariaDB: Modificar el `bind-address` a `0.0.0.0` mediante `lineinfile` en `/etc/mysql/mariadb.conf.d/50-server.cnf` para que escuche conexiones remotas .
3. Firewall (UFW Restrictivo): Habilitar el firewall de Ubuntu y autorizar conexiones entrantes al puerto `3306` (MariaDB) **únicamente** cuando provienen de la IP del servidor de aplicación CentOS (`10.0.2.15`) .
4. Creación e Inicialización de DB:
   - Crear la base de datos `cumples`.
   - Copiar e importar el dump inicial `files/cumpleanos.sql` de forma idempotente.
   - Crear el usuario `intranet` con contraseña encriptada, privilegios completos sobre la DB y origen de conexión autorizado desde cualquier host (`'%'`) o la IP específica del servidor CentOS.

---

## Verificación del Funcionamiento y Estado Deseado

Para validar el correcto despliegue e integración, verifique los siguientes puntos en sus respectivos entornos:

1. Prueba de Acceso Web (Criterio de Aceptación Principal):
   Acceda desde su navegador web o mediante terminal (vía `curl`) a la IP de su servidor CentOS Stream:

   curl -I http://10.0.2.15/
   
   *Debe responder un código de estado `HTTP/1.1 200 OK` y mostrar la tabla HTML con la lista de cumpleaños extraída desde la base de datos remota de Ubuntu.*
2. **Idempotencia (Obligatorio para la Entrega)**:
   Ejecute el playbook de despliegue por segunda vez consecutiva. La terminal de Ansible debe devolver que todas las tareas se encuentran en estado `ok` o `skipped`, con **`changed=0`** en el recap de ambos servidores gestionados.
3. **Firewall de Base de Datos**:
   En el nodo de base de datos Ubuntu, valide que la regla de UFW restrinja el puerto `3306` de MariaDB para que acepte solo la IP de CentOS:
   sudo ufw status verbose


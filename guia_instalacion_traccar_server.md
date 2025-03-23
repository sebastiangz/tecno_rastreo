# Guía de Instalación de Traccar en Fedora Linux

## Índice
1. [Descripción General](#descripción-general)
2. [Arquitectura de Componentes](#arquitectura-de-componentes)
3. [Requisitos Previos](#requisitos-previos)
4. [Instalación del Servidor](#instalación-del-servidor)
5. [Configuración de PostgreSQL](#configuración-de-postgresql)
6. [Configuración del Servidor Web](#configuración-del-servidor-web)
7. [Resolución de Problemas Comunes](#resolución-de-problemas-comunes)
8. [Configuración del Cliente Android](#configuración-del-cliente-android)
9. [Acceso a la Interfaz Web](#acceso-a-la-interfaz-web)
10. [Mantenimiento](#mantenimiento)

## Descripción General

Traccar es un sistema de seguimiento GPS de código abierto que admite más de 170 protocolos diferentes y permite rastrear dispositivos en tiempo real. Este README documenta los pasos verificados y funcionales para instalar Traccar en un servidor Fedora Linux, utilizando PostgreSQL como DBMS y Apache como servidor web frontal.

## Arquitectura de Componentes

### Componentes de Hardware

Para un despliegue efectivo de Traccar, se recomiendan los siguientes requisitos mínimos:

- **CPU**: 4+ cores
- **RAM**: 8+ GB
- **Almacenamiento**: 100+ GB SSD
- **Red**: 100+ Mbps

### Componentes de Software

La arquitectura de software de Traccar consta de:

- **Backend**:
  - Traccar Server (Java)
  - PostgreSQL (Base de Datos)
  - Apache HTTP Server (Proxy)

- **Frontend**:
  - Interfaz web basada en JavaScript/HTML5
  - API REST para integración con otros sistemas
  - WebSockets para actualizaciones en tiempo real

- **Clientes**:
  - Aplicación Android
  - Aplicación iOS
  - Interfaz web para navegadores

### Visualización de Componentes

```
                  +-------------------+
                  |   Cliente Web     |
                  | (HTML5/JavaScript)|
                  +--------+----------+
                           |
                           | HTTPS
                           |
+----------+      +--------v---------+      +-----------------+
|          |      |                  |      |                 |
| Cliente  +----->+ Apache HTTP      +----->+ Traccar Server  |
| Android  |      | (Proxy Reverse)  |      | (Java)          |
|          |      |                  |      |                 |
+----------+      +------------------+      +--------+--------+
                                                     |
                                                     | JDBC
                                                     |
                                              +------v-------+
                                              |              |
                                              | PostgreSQL   |
                                              | Database     |
                                              |              |
                                              +--------------+
```

## Requisitos Previos

### 1. Actualizar el Sistema
```bash
sudo dnf update -y
```

### 2. Instalar Java OpenJDK 11 o superior
```bash
sudo dnf install java-11-openjdk-devel -y
```

### 3. Verificar instalación de Java
```bash
java -version
```

### 4. Instalar Herramientas Necesarias
```bash
sudo dnf install wget unzip -y
```

## Instalación del Servidor

### Método 1: Instalación Automática (Recomendado)

1. **Descargar el instalador**:
```bash
wget https://github.com/traccar/traccar/releases/download/v4.16/traccar-linux-64-4.16.run
```

2. **Dar permisos de ejecución**:
```bash
chmod +x traccar-linux-64-4.16.run
```

3. **Ejecutar el instalador**:
```bash
sudo ./traccar-linux-64-4.16.run
```
Esto instalará Traccar en `/opt/traccar`

### Método 2: Instalación Manual

1. **Crear directorio de instalación**:
```bash
sudo mkdir -p /opt/traccar
sudo chown $USER:$USER /opt/traccar
```

2. **Descargar la última versión de Traccar**:
```bash
cd /opt/traccar
wget https://github.com/traccar/traccar/releases/download/v4.16/traccar-linux-64-4.16.zip
```

3. **Descomprimir el archivo**:
```bash
unzip traccar-linux-64-4.16.zip
```

## Configuración de PostgreSQL

1. **Instalar PostgreSQL**:
```bash
sudo dnf install postgresql postgresql-server postgresql-contrib -y
```

2. **Inicializar y habilitar PostgreSQL**:
```bash
sudo postgresql-setup --initdb
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

3. **Configurar PostgreSQL para Traccar**:
```bash
# Cambiar al usuario postgres
sudo -i -u postgres

# Crear usuario de base de datos para Traccar
psql -c "CREATE USER traccar WITH PASSWORD 'TuContraseñaSegura';"

# Crear base de datos para Traccar
psql -c "CREATE DATABASE traccar OWNER traccar;"

# Otorgar privilegios
psql -c "GRANT ALL PRIVILEGES ON DATABASE traccar TO traccar;"

# Salir del usuario postgres
exit
```

4. **Configurar PostgreSQL para permitir conexiones mediante contraseña**:
```bash
sudo vi /var/lib/pgsql/data/pg_hba.conf
```

Cambiar las líneas de autenticación de `ident` a `md5`:
```
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```

5. **Reiniciar PostgreSQL**:
```bash
sudo systemctl restart postgresql
```

6. **Configurar Traccar para usar PostgreSQL**:
```bash
sudo vi /opt/traccar/conf/traccar.xml
```

Modificar la sección de base de datos:
```xml
<entry key='database.driver'>org.postgresql.Driver</entry>
<entry key='database.url'>jdbc:postgresql://localhost:5432/traccar</entry>
<entry key='database.user'>traccar</entry>
<entry key='database.password'>TuContraseñaSegura</entry>
```

## Configuración del Servidor Web

1. **Instalar Apache (httpd) si no está instalado**:
```bash
sudo dnf install httpd mod_ssl mod_proxy_http mod_proxy_wstunnel -y
```

2. **Crear el archivo de configuración para el subdominio**:
```bash
sudo vi /etc/httpd/conf.d/rastreo.infraestructuragis.com.conf
```

Añadir el siguiente contenido:
```apache
<VirtualHost *:80>
    ServerName rastreo.infraestructuragis.com
    Redirect permanent / https://rastreo.infraestructuragis.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName rastreo.infraestructuragis.com
    
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/infraestructuragis.com.crt
    SSLCertificateKeyFile /etc/ssl/private/infraestructuragis.com.key
    SSLCertificateChainFile /etc/ssl/certs/infraestructuragis.com-chain.crt
    
    ProxyPreserveHost On
    ProxyRequests Off
    
    # Configuración de proxy para la API de Traccar
    ProxyPass /api/ http://localhost:8082/api/
    ProxyPassReverse /api/ http://localhost:8082/api/
    
    # Configuración de proxy para el socket WebSocket
    ProxyPass /api/socket ws://localhost:8082/api/socket
    ProxyPassReverse /api/socket ws://localhost:8082/api/socket
    
    # Configuración para la interfaz web de Traccar
    ProxyPass / http://localhost:8082/
    ProxyPassReverse / http://localhost:8082/
    
    ErrorLog /var/log/httpd/rastreo.infraestructuragis.com-error.log
    CustomLog /var/log/httpd/rastreo.infraestructuragis.com-access.log combined
</VirtualHost>
```

3. **Configurar SELinux para permitir que httpd actúe como proxy**:
```bash
sudo setsebool -P httpd_can_network_connect 1
```

4. **Reiniciar Apache**:
```bash
sudo systemctl restart httpd
```

## Configuración del Servicio Traccar

1. **Habilitar e iniciar el servicio**:
```bash
sudo systemctl enable traccar
sudo systemctl start traccar
```

2. **Verificar el estado del servicio**:
```bash
sudo systemctl status traccar
```

## Configuración de Firewall

1. **Abrir puertos necesarios en firewalld**:
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8082/tcp  # Puerto por defecto de Traccar
# Para dispositivos específicos
sudo firewall-cmd --permanent --add-port=5001-5050/tcp
sudo firewall-cmd --reload
```

## Resolución de Problemas Comunes

### Error: Unable to access jarfile

Si encuentras este error en los logs:
```
Error: Unable to access jarfile tracker-server.jar
```

**Solución**:
1. Verificar el nombre correcto del archivo JAR:
```bash
find /opt/traccar -name "*.jar"
```

2. Actualizar el archivo de servicio con el nombre correcto:
```bash
sudo vi /etc/systemd/system/traccar.service
```

3. Recargar y reiniciar:
```bash
sudo systemctl daemon-reload
sudo systemctl restart traccar
```

### Error: Connection refused

Si Apache no puede conectarse a Traccar:
```
Connection refused: AH00957: http: attempt to connect to 127.0.0.1:8082 (localhost:8082) failed
```

**Solución**:
1. Verificar que Traccar está ejecutándose:
```bash
sudo systemctl status traccar
```

2. Verificar los logs de Traccar:
```bash
sudo journalctl -u traccar -n 50
```

3. Reiniciar el servicio:
```bash
sudo systemctl restart traccar
```

### Error: WebSocket 405 Method Not Allowed

Si ves errores relacionados con WebSocket:
```
GET /api/socket HTTP/1.1" 405 99
```

**Solución**:
1. Verificar que mod_proxy_wstunnel está instalado:
```bash
sudo httpd -M | grep proxy_wstunnel
```

2. Asegurarse de que la configuración de Apache incluye correctamente:
```apache
ProxyPass /api/socket ws://localhost:8082/api/socket
ProxyPassReverse /api/socket ws://localhost:8082/api/socket
```

3. Reiniciar Apache:
```bash
sudo systemctl restart httpd
```

## Configuración del Cliente Android

1. **Instalar la aplicación**:
   - Descargar "Traccar Client" desde Google Play Store

2. **Configurar la aplicación**:
   - URL del servidor: `https://rastreo.infraestructuragis.com`
   - Identificador del dispositivo: Crear un identificador único
   - Frecuencia: 60 segundos (ajustar según necesidades)
   - Habilitar "Iniciar al encender" si se desea

3. **Iniciar el servicio**:
   - Presionar el botón "Iniciar" en la aplicación

4. **Si la aplicación muestra "Fallo en el envío"**:
   - Verificar que la URL sea correcta
   - Asegurarse de que el dispositivo tenga conexión a Internet
   - Verificar que el servidor esté funcionando correctamente
   - Probar tanto con URL terminada en `/api` como sin ella

## Acceso a la Interfaz Web

1. **Credenciales por defecto**:
   - **Usuario**: admin
   - **Contraseña**: admin

2. **Acceder a través de**:
   ```
   https://rastreo.infraestructuragis.com
   ```

3. **Cambiar la contraseña por defecto**:
   - Hacer clic en el ícono de usuario en la esquina superior derecha
   - Seleccionar "Preferencias"
   - Ir a la sección "Usuario"
   - Cambiar la contraseña

## Mantenimiento

### Respaldos de Base de Datos

Configurar respaldos automáticos:
```bash
sudo vi /etc/cron.daily/backup-traccar-db
```

Añadir:
```bash
#!/bin/bash
BACKUP_DIR="/home2/backups/traccar"
DATE=$(date +%Y-%m-%d)
mkdir -p $BACKUP_DIR
pg_dump -U traccar -h localhost traccar > $BACKUP_DIR/traccar-$DATE.sql
```

Dar permisos de ejecución:
```bash
sudo chmod +x /etc/cron.daily/backup-traccar-db
```

### Verificación de Logs

Para revisar los logs de Traccar:
```bash
sudo journalctl -u traccar -f
```

Para revisar los logs de Apache:
```bash
sudo tail -f /var/log/httpd/rastreo.infraestructuragis.com-error.log
```

### Actualizaciones

Para actualizar Traccar a una nueva versión:
1. Detener el servicio: `sudo systemctl stop traccar`
2. Descargar la nueva versión
3. Instalar siguiendo los pasos de instalación
4. Iniciar el servicio: `sudo systemctl start traccar`

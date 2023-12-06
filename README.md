# BalanceadorDeCargaConApache - Antonio Cruz Clavel - 06/12/2023
# Índice.
1. Esquema.
2. Descripción.
3. Creación VPC's
4. Creación instancias.
   * 4.1 Balanceador de carga.
   * 4.2 Servidores apache.
   * 4.3 BBDD MYSQL.
   * 4.4 Pivote ssh balancer.
5. Configuraciones.
   * 5.1 Balanceador de carga.
   * 5.2 Servidores Apache.
   * 5.3 BBDD MYSQL
6. Grupos de seguridad.
   * 6.1 Balanceador de carga.
   * 6.2 Servidores Apache.
   * 6.3 BBDD MYSQL.
7. Screencash

# 1. Esquema.
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Esquema%20proyecto.png)
# 2. Descripción.
Esta práctica se basa en el montaje de una pila LAMP en tres niveles con instancias EC2 de AWS.

Una pila LAMP es un conjunto de software de código abierto utilizado para desarrollar y ejecutar aplicaciones web. El acrónimo LAMP se refiere a los componentes principales de esta pila:

**Linux**: Es el sistema operativo base en el que se ejecutará la aplicación web. 

**Apache**: Es el servidor web que se encarga de gestionar las solicitudes HTTP de los clientes y entregar las páginas web correspondientes. 

**MySQL**: Es un sistema de gestión de bases de datos relacional  que se utiliza para almacenar y gestionar los datos de la aplicación.

**PHP**: Es un lenguaje de programación de código abierto diseñado específicamente para el desarrollo web.

Los diferentes niveles se disponen así:  

**Nivel 1**. Balanceador de carga.    
**Nivel 2**. Servidores web Apache.    
**Nivel 3**. Servidores BBDD MYSQL.  

# 3. Creación VPC's.

Una **VPC** se refiere a un servicio de computación en la nube que permite a los usuarios crear y gestionar redes privadas virtualizadas en entornos de nube pública.
Para crear una **VPC** en **AWS** tan solo se debe ir al servicio VPC y clickar en **CREAR VPC**. Para realizar el montaje de manera práctica y crear la subredes asociandola a nuesta VPC directamente usaremos la opción *VPC y más* en el siguiente menú.  

![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Configuración%20VPC%201.png)
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Configuración%20VPC%202.png)  

En dicho menú se necesitan modificar los siguientes parámetros:
* **Bloque CIDR IPv4**: 10.0.5.0/24.
* **Número de zonas de disponibilidad**: 1.
* **Cantidad de subredes públicas**: 1, para el balanceador de cargas.
* **Cantidad de subredes privadas**: 1, para el backend.
* **Personalizar bloques CIDR de subredes**: 10.0.5.0/25 y 10.0.5.128/25.
* **Gateway NAT**: en 1 AZ.

Así pues este es el esquema VPC obtenido:  

![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/VistapreviaVPCbuena.png)

# 4. Creación instancias.
 Las 4 instancias necesarias (1 instancia que actúe como balanceador de cargas, 2 máquinas apaches y 1 máquina BBDD MYSQL) se configurarán de la siguiente manera:
 ## 4.1 Balanceador de cargas.
 ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Instanciabalanceador1.png)

 ## 4.2 Servidores Apache  
 (Ambos se crean de la misma forma, cambiando solo la etiqueta)
 ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/instancia%20apache1nombre.png)
 ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/instanciapache1configuraciondered.png)  

 ## 4.3 BBDD MYSQL.
Se creará igual que los servidores apache asignándole la subred privada.  

Tal y como se muestra en las capturas anteriores los campos a modificar son los siguientes:
* **Nombre y etiquetas**
* **AMI**: Debian
* **Par de Claves**: Vockey
* **Configuración de red**: Se asignará la subred privada a los servidores de backend (apaches y BBDD SQL) y la subred pública al balanceador. No se le otorgará una IP pública temporal a las instancias.

## 4.4 Pivote ssh balancer.
Al no asignar una IP pública temporal a las instancias previamente se necesita una manera de acceder a ellas. Para ello en esta práctica se crea una instancia temporal llamada **"pivote ssh balancer"** que facilitará el movimiento entre las máquinas, por lo tanto a esta instancia si se le otorga una IP pública temporal.
  ### Comandos útiles.
  Para mover el certificado a la **máquina pivote**: `scp -i labsuser.pem labsuser.pem admin@10.0.5.4`  
  Cambiar nombre al hostname de las máquinas: `sudo hostnamectl set-hostname Apache1`  
  Cambiar los permisos y validar el certificado: `chmod go-r labsuser.pem`  

  # 5. Configuraciones.
  ## 5.1 Balanceador de carga.
  En primer lugar se le asigna una dirección IP pública elástica que previamente creamos, ya que el balanceador será la máquina por la que podamos entrar a nuestro servidor backend.
  Para ello tan solo hay que ir a **Direcciones IP elásticas** en el menú *Red y seguridad* y pinchar en *Asociar una dirección IP elástica*.
  *Dentro del menú no hará falta editar ningún parámetro*.   
  ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/ipelastica.png)  
   Luego habrá que asociar la IP elástica que hemos creado a la instancia **balanceador de carga**.  

   Una vez dentro de nuestra máquina hay que:
   * Instalar apache. `sudo apt install apache2`
   * Activar los módulos siguientes:
```
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_ajp
sudo a2enmod rewrite
sudo a2enmod deflate
sudo a2enmod headers
sudo a2enmod proxy_balancer
sudo a2enmod proxy_connect
sudo a2enmod proxy_html
sudo a2enmod lbmethod_byrequests
```
   * Crear el archivo balancer.conf, activar ssl y reiniciar el servicio apache2:
```
sudo cp default-ssl.conf balancer.conf
sudo a2ensite balancer.conf
sudo a2dissite 000-default.conf
sudo a2enmod ssl
sudo systemctl restart apache2
```
  * Editar proxy en balancer.conf de la siguiente manera:
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/balancerconf.png)


### 5.1.1 Certbot y Let's encrypt
Para crear un DNS se recomienda usar [My No-IP](https://www.noip.com/es-MX). En el caso de la práctica así ha sido creado:  
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/ddns.png)

Siguiendo en la máquina **balanceador de carga**, se debe instalar certbot con ayuda de snap:  
```
sudo apt update
sudo apt install snapd

sudo snap install core
sudo snap refresh core

sudo apt remove certbot
sudo snap install --classic certbot
```
Para configurar certbot con el DNS anteriormente creado:
`sudo certbot --apache`
La instalación del certificado pedirá datos como el correo electrónico, aceptar términos de uso y el nombre de dominio.  
**IMPORTANTE** Tener activo un archivo de configuración http para validar el control del dominio, si no lo encuentra dará error.  

Cuando aparezca el mensaje *Congratulations! You have successfully enabled HTTPS on https://antoniocruz.ddns.net* el proceso habrá terminado y el certificado habrá sido instalado.
   
  ## 5.2 Servidores Apache.
Instalar apache y php:
   ```
sudo apt update
sudo apt install -y apache2
sudo apt install php libapache2-mod-php php-mysql
```
Crear usuarios.conf (fichero de configuración de la aplicación de usuarios).: 
```
cp 000-default.conf usuarios.conf
sudo nano usuarios.conf
```
Editar documentroot para que sea el de la aplicación de usuarios
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/configuracion%20apache%20usuarios.conf.png)

Activar el archivo de configuración anterior y desactivar la plantilla:
```
sudo a2ensite usuarios.conf
sudo a2dissite 000-default.conf
```

Editar config.php con los parámetros que posteriormente se le darán en el servidor de base de datos:
```
sudo nano /var/www/html/usuarios/src/config.php
```
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/configuracion%20apache1%20usuariossrcconfigphp.png)

Reiniciar el servicio apache2:
```
sudo systemctl restart apache2
```
Instalar mariadb-client:  
```
sudo apt install -y mariadb-client
```
## 5.3 BBDD MYSQL
Instalar mariadb-server.  

```
sudo apt update
sudo apt install -y mariadb-server
```
Modificar el parámetro bind address para indicar a que dirección llegarán las peticiones SQL [indicamos la ip privada del servidor mysql (10.0.5.133 a la cual podrán llegar solicitudes de cualquier ip dentro de su red (10.0.5.0)]
`sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf`
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/bind%20address.png)

Cargar el script database.sql que previamente hemos traido al servidor con *wget* o *gitclone*:  
```
sudo mysql -u root < database.sql
sudo mysql -u root
CREATE USER 'antonio@10.0.10.%' IDENTIFIED BY 'cruz';
GRANT ALL PRIVILEGES ON lamp_db.* TO 'usuarios_user@10.0.5.%';
FLUSH PRIVILEGES;
```
Comprobar que se puede acceder desde una maquina apache con el usuario antonio al servidor MYSQL:
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/comprobacionconexionapachesql.png)


**IMPORTANTE**: CUIDADO AL USAR carácteres especiales en el usuario/contraseña ya que estos dan problemas a la hora de conectarse al servidor MYSQL de forma remota.

# 6. Grupos de seguridad
## 6.1 Balanceador de carga.
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/GSBalanceador.png)  
Puerto 443 abierto para comunicarse con internet y puerto 80 abierto para comunicarse con los servidores apache.  
## 6.2 Servidores Apache. 
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/GSServidoresApache.png)
Puerto 80 abierto para comunicarse con el balanceador y puerto 3306(MYSQL) abierto para comunicarse con la BBDD.
## 6.3 BBDD MYSQL. 
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/GSBBDD.png)
Puerto 3306(MYSQL) abierto.

# 7. Screencash
[Pincha aquí para obtener el screencash](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Screencash.mp4) donde se aprecia el funcionamiento de la aplicación desplegada. 


# BalanceadorDeCargaConApache - Antonio Cruz Clavel - 06/12/2023
Proyecto IAW ASIR2 IES ALBARREGAS con un balanceador de cargas, 2 máquinas backend apache y 1 SQL con Amazon AWS EC2
# 1. Esquema
![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Esquema%20proyecto.png)
# 2. Descripción
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
 ## Balanceador de cargas
 ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/Instanciabalanceador1.png)

 ## Servidores Apache  
 (Ambos se crean de la misma forma, cambiando solo la etiqueta)
 ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/instancia%20apache1nombre.png)
 ![](https://github.com/acruzc05/BalanceadorDeCargaConApache/blob/main/instanciapache1configuraciondered.png)  

 ## BBDD MYSQL
Se creará igual que los servidores apache asignándole la subred privada.  

Tal y como se muestra en las capturas anteriores los campos a modificar son los siguientes:
* **Nombre y etiquetas**
* **AMI**: Debian
* **Par de Claves**: Vockey
* **Configuración de red**: Se asignará la subred privada a los servidores de backend (apaches y BBDD SQL) y la subred pública al balanceador. No se le otorgará una IP pública temporal a las instancias.

## Pivote ssh balancer
Al no asignar una IP pública temporal a las instancias previamente se necesita una manera de acceder a ellas. Para ello en esta práctica se crea una instancia temporal llamada **"pivote ssh balancer"** que facilitará el movimiento entre las máquinas, por lo tanto a esta instancia si se le otorga una IP pública temporal.
  ### Comandos útiles
  Para mover el certificado a la **máquina pivote** `scp -i labsuser.pem labsuser.pem admin@10.0.5.4`  
  Cambiar nombre al hostname de las máquinas `sudo hostnamectl set-hostname Apache1`  
  Cambiar los permisos y validar el certificado `chmod go-r labsuser.pem`  
  


---
layout: single
title: Return Writeup HTB (OSCP Style)
excerpt: "En el siguiente artículo veremos como realizar una enumeración al mome
nto de estar en un pentesting sobre windows"
date: 2022-05-07
classes: wide
image: /assets/img/posts/HTB/driver/driver.png
categories:
  - HTB
tags:
  - Hacking
---
![header-return](/assets/img/posts/HTB/return/return.png)

## Datos de la Máquina

| Contenido | Descripción |                                                                                                                                        
|--|--|                                                                                                                                                            
| OS: | ![windows](/assets/img/posts/windows.png) |                                                                                                                                         
| Dificultad: | Fácil |                                                                                                                                            
| Puntos: | 20 |                                                                                                                                                   
| Lanzamiento: | 27 Sep 2021 |                                                                                                                                
| Creadores: |[MrR3boot](https://app.hackthebox.com/users/13531) |

## Reconocimiento

Primero agregaremos nuestra ip al archivo **/etc/hosts** con el dominio **return.htb**

![hosts](/assets/img/posts/HTB/return/hosts.png) 

### nmap

Como siempre comenzaremos con un escaneo a los **65535** puertos **TCP/IP** para verificar cuales se encuentran abiertos, para esto ejecutaremos el siguiente comando:
```console
nmap -n -p- -T4 --min-rate 10000 return.htb --open -oA alltcp
```
Como observamos en la siguiente imagen encontramos con los siguientes puertos abiertos:
- 53 (domain)
- 80 (http)
- 135 (msrpc)
- 139 (netbios-ssn)
- 445 (microsfot-ds)
- 464 (kpasswd5)
- 593 (http-rpc-epmap)
- 3269 (globalcatLDAPssl)
- 49675  (unknown)

![nmap-tcpscan](/assets/img/posts/HTB/return/nmap-tcpscan.png)

Una vez identificados los puertos abiertos, procederemos a identificar las versiones y servicios de cada uno, y de igual manera arrojaremos los script por defecto de nmap:  
  
Cabe mencionar que el argumento **-sCV** es una union de usar **-sV** y **-sC**

```console
nmap -sCV -n -p 53,80,135,139,445,464,593,3269,49675 -T4 return.htb -oA tcp-scripts
```
![nmap-scripts](/assets/img/posts/HTB/return/nmap-scripts.png)

Al identificar una máquinas Windows con varios puertos abiertos, podemos hacer uso de la herramienta **Crackmapexec**, para confirmar el hostname.

![cme](/assets/img/posts/HTB/return/cme.png)

Haciendo una busqueda por internet encontre una guía para recopilar información usando Python3: [Hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-ldap) 

![python-ldap](/assets/img/posts/HTB/return/python-ldap.png)

Agregamos el host en nuestro **etc/hosts**

```console
echo '10.129.132.85	return.local' >> /etc/hosts
```
### Printer Admin

Observando los puertos abiertos que encontramos en la fase de reconocimiento nos percatamos que teniamos el puerto **80** abierto, lo cual visitamos en el navegador y nos aparece un panel de administración de impresion.

![printer-panel](/assets/img/posts/HTB/return/printer-admin-panel.png)

Navegamos sobre la web y encontramos información relevante en la configuración

![settings-printer](/assets/img/posts/HTB/return/settings-printer.png)

Observando la contraseña se encuentra cifrada, por lo cual buscando en la red nos escontramos con [este](https://www.ceos3c.com/security/obtaining-domain-credentials-printer-netcat/) articulo lo cual podemos interceptar la contraseña en texto plano con netcat.

![nc-389](/assets/img/posts/HTB/return/nc-389.png)

### Crackmapexec

Ahora que tenemos las credenciales, trataremos de obtener más información con Crackmapexec.
![cmd-cred](/assets/img/posts/HTB/return/cme-cred.png)

### SMBMap

De igual forma podemos conectarnos ahora que ya tenemos credenciales.
```console
smbmap -H return.local -u "svc-printer" -p "1edFg43012\!\!"
```
![smbmap-01](/assets/img/posts/HTB/return/smbmap-01.png)

Observamos que tenemos permisos de **(Lectura y Escritura)** sobre el disco C.
```console
smbmap -H return.local -u "svc-printer" -p "1edFg43012\!\!" -r C$
```
![smbmap-02](/assets/img/posts/HTB/return/smbmap-02.png)

Nos vamos a la carpeta Usuarios para darle un vistazo.
```console
smbmap -H return.local -u "svc-printer" -p "1edFg43012\!\!" -r C$/Users/
```
![smbmap-03](/assets/img/posts/HTB/return/smbmap-03.png)

Nos dirigos a el usuario con el cual iniciamos sesión.
```console
smbmap -H return.local -u "svc-printer" -p "1edFg43012\!\!" -r C$/Users/svc-printer
```
![smbmap-04](/assets/img/posts/HTB/return/smbmap-04.png)

Hechamos un vistazo a las carpetas y nos vamos sobre **Desktop**
```console
smbmap -H return.local -u "svc-printer" -p "1edFg43012\!\!" -r C$/Users/svc-printer/Desktop
```
![smbmap-05](/assets/img/posts/HTB/return/smbmap-05.png)

Tenemos la primer bandera de **user.txt**, ahora la descargamos sobre nuestra máquina.
```console
smbmap -H return.local -u "svc-printer" -p "1edFg43012\!\!" -r C$/Users/svc-printer/Desktop -A "user.txt"
```
![smbmap-06](/assets/img/posts/HTB/return/smbmap-06.png)

### Evil-WinRM

Con las credenciales que obtuvimos anteriormente de igual forma podremos obtener una shell interactiva usando [Evil-WinRM](https://github.com/Hackplayers/evil-winrm)  
![shell-evil-winrm](/assets/img/posts/HTB/return/shell-evilwinrm.png)

### Recopilación de Información del Usuario
Ahora que tenemos una shell interactiva podemos investigar, usuarios, grupos y permisos.
- whoami
- whoami /groups
- net users
- net groups 
- net usr svc-printer  
![evilwinrm-enum-01](/assets/img/posts/HTB/return/evilwinrm-enum-01.png)
![evilwinrm-enum-02](/assets/img/posts/HTB/return/evilwinrm-enum-02.png)
![evilwinrm-enum-03](/assets/img/posts/HTB/return/evilwinrm-enum-03.png)

Arriba hemos recopilado mucha información útil, lo más interesante que vemos es que nuestro usuario está en los grupos de Operadores del servidor. Detallado [aquí](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#bkmk-serveroperators) vemos que esto nos da:

```console
Members of the Server Operators group can sign in to a server interactively,
create and delete network shared resources, start and stop services,
back up and restore files, format the hard disk drive of the computer,
and shut down the computer.
```
### Configuración de Servicios

El enfoque de nuestro próximo paso es interactuar con los servicios. Podemos ver desde dentro de Evil-WinRM a qué tenemos acceso:  
![evilwinrm-services](/assets/img/posts/HTB/return/evilwinrm-services.png)

Primero tendremos que subir a Windows la versión de netcat.
![nc-upload](/assets/img/posts/HTB/return/evilwinrm-01.png)

Despues de subirlo trataremos de crear nuestro propio servicio:
![own-service](/assets/img/posts/HTB/return/evilwinrm-02.png) 

Luego intenté cambiar la configuración de un servidor existente para que use mi netcat en su lugar para su binario:
![config-service](/assets/img/posts/HTB/return/evilwinrm-03.png)

Finalmente obtenemos el servicios VSS y anclamos nuestro binario a este:
![nc-vss](/assets/img/posts/HTB/return/evilwinrm-04.png)

Verificamos que este servicio exista:
![service-vss](/assets/img/posts/HTB/return/evilwinrm-05.png)

Antes de iniciarlo tendremos que poner a la escuche netcat en lel puerto especificado:

```console
nc -nvlp 4444
```

Iniciamos el servicio:

![create-service](/assets/img/posts/HTB/return/evilwinrm-06.png)

### Obtenemos Shell de Administrador

Una vez iniciado el servicio nos arrojara una shell en netcat.
![create-service](/assets/img/posts/HTB/return/evilwinrm-07.png)

Ahora unicamente tendremos que encontrar nuestra bandera de root:  
  
Cabe mencionar que las acciones las tendremos que hacer rápido ya que en cierto tiempo de termina el servicio y tendremos que volver iniciar netcat junto con el servicio:
![create-service](/assets/img/posts/HTB/return/evilwinrm-08.png)

Good Luck! and Happy Hacking!

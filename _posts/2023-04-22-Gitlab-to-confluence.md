---
title: Integrar API gitlab a confluence.
date: 2023-04-22
categories: [API]
tags: [Linux, Windows, Gitlab, Confluence]
image:
  path: /assets/img/gitlab-to-confluence/gitlab.png
  width: 500 #normal 800
  height: 500 #normal 500
  alt: gitlab-to-confluence
---

# Cómo integrar Confluence con GitLab.

En este manual, aprenderemos a integrar dos tecnologías a través de sus API utilizando Python. El laboratorio que vamos a realizar muestra cómo obtener los datos de un README.md de algún proyecto de GitLab y subir ese contenido a Confluence en un espacio y página previamente configurado con el mismo formato de origen.

## Configuración.

Crear y configurar el laboratorio.

Requerimientos.
- Cuenta en GitLab.
- Cuenta en Confluence.
- Python.

---

### GitLab.

1. Crear una cuenta en GitLab.
2. Creamos un proyecto.
3. Generamos un token para el proyecto.

Con Python, vamos a configurar una conexión a la API de GitLab y ver qué opciones tenemos para conectar y ver el contenido de un proyecto.

Requerimientos.
- Librería de GitLab: `pip install python-gitlab`.
- Librería Markdown: `pip install markdown2`.
- Nombre de usuario.
- ID del proyecto.
- Token.

````python
import gitlab

#Conexion a gitlab
gl = gitlab.Gitlab('https://gitlab.com', private_token='TU-TOKEN')

# Obtener datos del proyecto.
project_id = 'PROYECTO-ID'
project = gl.projects.get(project_id)

print(project)

#Respuesta.
<class 'gitlab.v4.objects.projects.Project'>

````

Con esta conexión, podemos comprobar la configuración de GitLab. Pero ahora vamos a obtener información más detallada del proyecto, como el ID, la hora y el correo electrónico.

````python
import gitlab

#Conexion a gitlab
gl = gitlab.Gitlab('https://gitlab.com', private_token='TU-TOKEN')

# Obtener datos del proyecto.
project_id = 'PROYECTO-ID'
project = gl.projects.get(project_id)

# Obtener el último commit
commit = project.commits.list()[0]

print(f"ID:{commit.id} \nFECHA:{commit.authored_date} \nEMAIL:{commit.committer_email}")

#Respuesta
ID:87a993b0379ff87e0b1832684d36c077ff528075 
FECHA:2023-04-19T13:36:37.000+00:00
EMAIL:test@gmail.com
````

>Puedes ver mas opciones llamando solo la variable commit `print(commit)`

Vemos que podemos obtener información más detallada de los commits de un proyecto, pero el objetivo de este laboratorio es poder obtener la información que contiene el README.md. Vamos a seguir desarrollando el script para lograrlo.

````python
import gitlab
import markdown2

#Conexion a gitlab
gl = gitlab.Gitlab('https://gitlab.com', private_token='TU-TOKEN')

# Obtener datos del proyecto.
project_id = 'PROYECTO-ID'
project = gl.projects.get(project_id)

# Obtener el último commit
commit = project.commits.list()[0]

# Obtiene el árbol de archivos del proyecto
tree = project.repository_tree()

# Itera sobre el árbol de archivos y muestra el contenido de cada archivo
for file in tree:

    # Verifica si el archivo es un archivo README.md
    if file["name"].lower() == "readme.md":
        # Obtiene el contenido del archivo
        file_content = project.files.get(file["path"], ref=commit.id).decode()
# Imprime el contenido del archivo
html = markdown2.markdown(file_content)
print(html)

#Respuesta

<h1>CONTENIDO DEL README</h1>

<p>Este readme.md es un ejemplo.</p>

<h2>Subtítulo</h2>

<h3>Notas</h3>
````

En este punto, ya tenemos acceso al contenido del README.md. Ahora vamos a configurar el entorno de Confluence para subir la información que obtuvimos de GitLab.

### Confluence

1. Crear una cuenta en Confluence.
2. Crear un espacio.
3. Crear una página.
4. Generar un token.

Como hicimos anteriormente con GitLab, vamos a crear una conexión para hacer pruebas y ver qué opciones tenemos.

Requerimientos:
- Librería: `pip install atlassian-python-api`.
- Librería Markdown: `pip install markdown2`.
- Nombre de usuario.
- Token.
- Nombre del espacio.
- Nombre de la página. 

````python
from atlassian import Confluence

url = 'https://TEST.atlassian.net'
username = 'TU-USUARIO'
token = 'TU-TOKEN'

# Conectar Confluence
confluence = Confluence(url=url, username=username, password=token)

# Selecciones su espacio y titulo de la pagina
space_key = 'TU-ESPACIO'
page_title = 'TU-PAGINA'

# Verificar que existe la pagina
existing_page = confluence.get_page_by_title(space=space_key, title=page_title)

#Respuesta.
print(existing_page)
>'id': '6545875234', 'type': 'page', 'status': 'current'
>Error al conectarse a Confluence. #Comprueba los datos de las variables url, username, token
````

Si recibimos una respuesta exitosa con un status code 200, ya podemos realizar pruebas y subir contenido a Confluence. Vamos a probar subiendo un poco de texto a la página que creamos.

````python
from atlassian import Confluence

url = 'https://TEST.atlassian.net'
username = 'TU-USUARIO'
token = 'TU-TOKEN'

# Conectar Confluence
confluence = Confluence(url=url, username=username, password=token)

# Selecciones su espacio y titulo de la pagina
space_key = 'TU-ESPACIO'
page_title = 'TU-PAGINA'

# Verificar que existe la pagina
existing_page = confluence.get_page_by_title(space=space_key, title=page_title)

page_id = existing_page['id']
current_content = confluence.get_page_by_id(page_id=page_id, expand='body.storage')['body']['storage']['value']

#Contenido que vamos a agregar.
add_text = "CONTENIDO DE PRUEBA"

#Actualizamos la pagina con el contenido nuevo.
updated_content = f"{add_text}"
confluence.update_page(page_id=page_id, title=page_title, body=updated_content)

````

Verificamos en Confluence el contenido que agregamos a la página.

![imagen-de-prueba](/assets/img/gitlab-to-confluence/img1.png)


En este punto, ya tenemos una forma de subir contenido a Confluence, pero tenemos un problema: cada vez que subimos contenido, eliminamos el anterior. Esto es válido, pero el objetivo de este laboratorio es almacenar un historial de los cambios del proyecto de GitLab.

### Integración de GitLab con Confluence

Vamos a cambiar el contenido que subimos a Confluence por información obtenida de GitLab. Comencemos subiendo los datos de un commit, como el ID, el correo electrónico y la fecha.

````python
import gitlab
import markdown2
from datetime import datetime

#Conexion a gitlab
gl = gitlab.Gitlab('https://gitlab.com', private_token='TU-TOKEN')

# Obtener datos del proyecto.
project_id = 'PROYECTO-ID'
project = gl.projects.get(project_id)

# Obtener el último commit
commit = project.commits.list()[0]

##############################################################################################
# Convertir a markdown
date = datetime.now()

new_md = f"\n --- \n# Informe fecha {date.date()} \n**Hora** *{date.strftime('%H:%M')}* \n**ID**: *{commit.id}* \n**Mail**: *{commit.committer_email}* \n**Mensaje**: *{commit.title}* \n ---"

new_html = markdown2.markdown(new_md)

##############################################################################################

from atlassian import Confluence

url = 'https://TEST.atlassian.net'
username = 'TU-USUARIO'
token = 'TU-TOKEN'

# Conectar Confluence
confluence = Confluence(url=url, username=username, password=token)

# Selecciones su espacio y titulo de la pagina
space_key = 'TU-ESPACIO'
page_title = 'TU-PAGINA'

# Verificar que existe la pagina
existing_page = confluence.get_page_by_title(space=space_key, title=page_title)

page_id = existing_page['id']
current_content = confluence.get_page_by_id(page_id=page_id, expand='body.storage')['body']['storage']['value']

#Actualizamos la pagina con el contenido nuevo.
updated_content = f"{current_content}\n{new_html}\"
confluence.update_page(page_id=page_id, title=page_title, body=updated_content)
````

![imagen-de-prueba](/assets/img/gitlab-to-confluence/img4.png)


> Se puede evitar usar la libreria datetime y usar `commit.authored_date` para obtener la fecha y la hora.

Como se ve en la imagen, hemos hecho dos llamadas a Gitlab y subido las respuestas a Confluence con formato Markdown. El script está cumpliendo el objetivo de integrar las API de Confluence y Gitlab.

Ahora vamos a ejecutar una prueba más, agregando el contenido del archivo README.md a la llamada anterior.

````python
import gitlab
import markdown2
from datetime import datetime

#Conexion a gitlab
gl = gitlab.Gitlab('https://gitlab.com', private_token='TU-TOKEN')

# Obtener datos del proyecto.
project_id = 'PROYECTO-ID'
project = gl.projects.get(project_id)

# Obtener el último commit
commit = project.commits.list()[0]

# Obtiene el árbol de archivos del proyecto
tree = project.repository_tree()

# Itera sobre el árbol de archivos y muestra el contenido de cada archivo
for file in tree:

    # Verifica si el archivo es un archivo README.md
    if file["name"].lower() == "readme.md":

        # Obtiene el contenido del archivo
        file_content = project.files.get(file["path"], ref=commit.id).decode()

        # Imprime el contenido del archivo
html = markdown2.markdown(file_content)

##############################################################################################

# Convertir a markdown
date = datetime.now()

new_md = f"\n --- \n# Informe fecha {date.date()} \n**Hora** *{date.strftime('%H:%M')}* \n**ID**: *{commit.id}* \n**Mail**: *{commit.committer_email}* \n**Mensaje**: *{commit.title}* \n ---"

new_html = markdown2.markdown(new_md)

##############################################################################################

from atlassian import Confluence

url = 'https://TEST.atlassian.net'
username = 'TU-USUARIO'
token = 'TU-TOKEN'

# Conectar Confluence
confluence = Confluence(url=url, username=username, password=token)

# Selecciones su espacio y titulo de la pagina
space_key = 'TU-ESPACIO'
page_title = 'TU-PAGINA'

# Verificar que existe la pagina
existing_page = confluence.get_page_by_title(space=space_key, title=page_title)

page_id = existing_page['id']
current_content = confluence.get_page_by_id(page_id=page_id, expand='body.storage')['body']['storage']['value']

#Actualizamos la pagina con el contenido nuevo.
updated_content = f"{current_content}\n{new_html}\n{html}"
confluence.update_page(page_id=page_id, title=page_title, body=updated_content)
````

![imagen-de-prueba](/assets/img/gitlab-to-confluence/img7.png)


Confirmamos que el contenido se ha subido correctamente a Confluence. De esta forma, podemos crear un historial muy completo en Confluence cada vez que se modifique un proyecto en GitLab.

Una cuestión a tener en cuenta es que el script se tiene que ejecutar de manera manual. Esto puede ser un problema si en algún momento realiza un cambio en GitLab y no recuerda ejecutar el script para guardar los cambios en Confluence. Se puede automatizar el script con una tarea programada en Windows o Linux. Esto guardaría datos, se hagan o no cambios en GitLab. Sin embargo, esto no tiene sentido si lo que queremos es guardar información solo cuando se realicen cambios en los proyectos. Así que vamos a automatizar todo esto usando webhooks y Flask.

## Automatizar el script.

### Configurar el webhook en GitLab.

Para configurar el webhook necesitamos Ngrok, una herramienta de uso gratuito que nos permite exponer nuestro entorno local a la web. Simplemente ejecutamos `ngrok.exe http 5000` y copiamos la URL que se genera en GitLab.

![imagen-de-prueba](/assets/img/gitlab-to-confluence/img3.png)


![imagen-de-prueba](/assets/img/gitlab-to-confluence/img5.png)


Ahora vamos a integrar Flask al script para crear un servidor web y escuchar los eventos del webhook de GitLab.

````python
from flask import Flask, request
from atlassian import Confluence
import gitlab
import markdown2
from datetime import datetime

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])

def webhook():

	#Conexion a gitlab
	gl = gitlab.Gitlab('https://gitlab.com', private_token='TU-TOKEN')
	
	# Obtener datos del proyecto.
	project_id = 'PROYECTO-ID'
	project = gl.projects.get(project_id)
	
	# Obtener el último commit
	commit = project.commits.list()[0]
	
	# Obtiene el árbol de archivos del proyecto
	tree = project.repository_tree()
	
	# Itera sobre el árbol de archivos y muestra el contenido de cada archivo
	for file in tree:
	
	    # Verifica si el archivo es un archivo README.md
	    if file["name"].lower() == "readme.md":
	
	        # Obtiene el contenido del archivo
	        file_content = project.files.get(file["path"], ref=commit.id).decode()
	
	        # Imprime el contenido del archivo
	html = markdown2.markdown(file_content)
	
	##############################################################################################
	
	# Convertir a markdown
	date = datetime.now()
	
	new_md = f"\n --- \n# Informe fecha {date.date()} \n**Hora** *{date.strftime('%H:%M')}* \n**ID**: *{commit.id}* \n**Mail**: *{commit.committer_email}* \n**Mensaje**: *{commit.title}* \n ---"
	
	new_html = markdown2.markdown(new_md)
	
	##############################################################################################
	
	from atlassian import Confluence
	
	url = 'https://TEST.atlassian.net'
	username = 'TU-USUARIO'
	token = 'TU-TOKEN'
	
	# Conectar Confluence
	confluence = Confluence(url=url, username=username, password=token)
	
	# Selecciones su espacio y titulo de la pagina
	space_key = 'TU-ESPACIO'
	page_title = 'TU-PAGINA'
	
	# Verificar que existe la pagina
	existing_page = confluence.get_page_by_title(space=space_key, title=page_title)
	
	page_id = existing_page['id']
	current_content = confluence.get_page_by_id(page_id=page_id, expand='body.storage')['body']['storage']['value']
	
	#Actualizamos la pagina con el contenido nuevo.
	updated_content = f"{current_content}\n{new_html}\n{html}"
	confluence.update_page(page_id=page_id, title=page_title, body=updated_content)
    return 'Webhook received and processed'

if __name__ == '__main__':

    app.run()
````

![imagen-de-prueba](/assets/img/gitlab-to-confluence/img6.png)


Confirmamos que cada vez que se genera un evento en el proyecto de GitLab, se envía una carga HTTP al servidor que creamos mediante Flask y, a continuación, el contenido se sube a Confluence.


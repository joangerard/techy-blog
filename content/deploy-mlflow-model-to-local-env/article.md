
---
title: "Deploy an MLflow model to local environment"
date: 2023-12-14T14:48:27+01:00

categories: ['Azure','ML', 'Tutorials']
tags: ['ML', 'Azure ML Studio', 'Azure', 'Tutorials']
author: "Joan Gerard"
---


# Introducción

Muchas veces lo que se quiere es hacer el desplegue (deploy) de un
modelo en otro entorno que no sea Azure debido a los altos costos que
esto puede incurrir. En esta guía se mostrará cómo se puede Desplegar un
modelo de manera local y poderlo dockerizar utilizando el artefacto
creado por Azure AutoML.

# Descargar el Artefacto

Un \"artefacto\" es un componente que resulta luego de haber entrenado
un modelo. El artefacto contiene el modelo entrenado junto con las
librerías necesarias para reutilizarlo. Dicho artefacto es el modelo que
está listo para ser Despliegado a un ambiente de producción.

Para este caso utilizaremos el artefacto creado para resolver la tarea
de multietiquetado.

Para descargarse dicho artefacto localmente, es necesario ya tenerlo en
nuestro entorno de Azure.


{{ $image := .Resources.Get "artifact-azure.png"}}


Lo comprimimos con este código ejecutado dessde nuestro notebook en
Azure ML:

```
    import os
    import zipfile
    
    folder_to_zip = "./artifact_downloads/outputs/mlflow-model"
    output_zip_file = "artifact_downloads.zip"
    
    with zipfile.ZipFile(output_zip_file, 'w') as zipf:
        for root, dirs, files in os.walk(folder_to_zip):
            for file in files:
                file_path = os.path.join(root, file)
                arcname = os.path.relpath(file_path, folder_to_zip)
                zipf.write(file_path, arcname)
```

Y lo descargamos en nuestro ordenador:


{{ $image := .Resources.Get "descargar.png"}}


Lo movemos a un directorio que crearemos llamado
`blog-multiclass-artifact` y lo descomprimimos en una carpeta
llamada `my_model` como se muestra a continuación.


{{ $image := .Resources.Get "directorio-ejemplo.png"}}


# Instalación de MLflow

MLflow es una herramienta de código abierto que nos permite gestionar
flujos de trabajo y artefactos para todo el ciclo de vida de un modelo.
Para encontrar más información [acceder
aquí](https://mlflow.org/docs/latest/index.html).

Es importante mencionar que a la actualidad, MLflow es compatible con
versiones menores o iguales a Python 3.10. Como requisito tendremos que
tener instalado Python en nuestro entorno. En mi caso tengo instalado
Python 3.10 de manera global.

Abrir una terminal, crear un entorno virtual de Python, e instalar
MLflow. Alternativamente se puede crear un entorno virtual con conda.
Notar que se instala la misma versión que usó Azure para crear el
artefacto, esta información se encuentra en el archivo MLmodel. En mi
caso estos comandos fueron ejecutados en MacOS 10.14.

``` 
    $ cd blog-multiclass-artifact
    $ python --version # 3.10
    $ python -m venv venv 
    $ source venv/bin/activate # en windows: venv/bin/activate.sh
    $ pip install mlflow==2.4.2
```

MLflow requiere tener instalado `virtualenv`:

``` 
    $ pip install virtualenv
```

MLflow requiere tener instalado `pyenv` (para Windows, Linux,
MacOS) y `libomp` (MacOS o Linux) para su correcto
funcionamiento. Ambos paquetes se instalan mediante el manejador de
paquetes `Homebrew`. [Click aquí](https://brew.sh/) para ver
cómo instalar Homebrew. Una vez instalado, abrir una terminal y
ejecutar:

``` 
    $ brew install pyenv
```

Para ver cómo instalar pyenv en Windows, [click
aquí](https://github.com/pyenv/pyenv).

Para instalar limbomp:

``` 
    $ brew install libomp
```

# Desplegar el modelo en local

Para Desplegar el modelo en local simplemente tenemos que ejecutar el
siguiente comando en una terminal dentro del directorio
`blog-multiclass-artifact`:

```
    $ mlflow models serve -m my_model --port 65321
```

MLflow buscará un directorio llamado `my_model` que contiene el
archivo `MLmodel` con la ruta del modelo en formato pickle
junto con el entorno a ser instalado (`python_env.yaml`).


{{ $image := .Resources.Get "MLmodel.png"}}


El archivo `python_env.yaml` contiene información sobre qué
versión de python se debe usar y qué librerías se debe instalar para
realizar el build y las dependencias del proyecto (requirements.txt).


{{ $image := .Resources.Get "python_env.png"}}


Asegurarse que el puerto usado, en este caso el 65321, se encuentra
disponible y no está siendo usado por otra aplicación. Si este fuera el
caso cambiarlo por otro número entre el 5000 y 65535.

Al ejecutar el comando debería ver esta respuesta indicando que el
endpoint está esperando requests:


{{ $image := .Resources.Get "corrida-exitosa.png"}}


# Realizar Inferencias

Para realizar una inferencia, basta con enviar un request POST a
`localhost:65321/invocations` con los datos que queremos que
etiquete. Por simplicidad podemos ussar una llamada curl desde la
terminal como se muestra a continuación:

``` 
    $ curl -d '{"dataframe_split": { "columns": ["titles", "summaries"], \
    “data": [{"titles": "some title", "summaries": "price is \ 
    reasonable"}] }}' -H 'Content-Type: application/json' \
    -X POST localhost:65321/invocations
```

Y obtenemos las etiquetas para el texto:


{{ $image := .Resources.Get "curl-command.png"}}


# Desplegar un modelo en un contenedor Docker

[Docker](https://docs.docker.com/get-started/overview/) es una
plataforma de código abierto que posibilita el desarrollo, la entrega y
la ejecución de aplicaciones. Con Docker, es posible separar las
aplicaciones de la infraestructura, lo que facilita la entrega ágil de
software. Esta herramienta permite gestionar la infraestructura de la
misma manera en que se gestionan las aplicaciones. Al aprovechar las
prácticas de Docker para Desplegar, probar e implementar código, se
puede reducir de manera sustancial el intervalo de tiempo entre la
escritura del código y su puesta en producción.

Descargar e instalar [Docker
Desktop](https://www.docker.com/products/docker-desktop/) y habilitar
los features experimentales:


{{ $image := .Resources.Get "docker-features.png"}}


Ejecutar el siguiente comando desde terminal:

``` 
    $ mlflow models build-docker --model-uri my_models --name \
    “blog_multi_class_container”
```

Este comando crea una imagen Docker llamada
**blog_multi_class_container** que contiene el modelo y todas sus
dependencias. Una vez terminado de ejecutar, se puede realizar
inferencias localmente, on-premises, en un server o en una plataforma
cloud. Para correrlo localmente:

``` 
    $ docker run -p 5002:8080 blog_multi_class_container
```

Este comando mapea el puerto 5002 en la máquina local al puerto 8080 en
el contenedor. Para enviar requests se puede utilizar el mismo comando
anterior:

``` 
    $ curl -d '{"dataframe_split": { "columns": ["titles", "summaries"], \
    “data": [{"titles": "some title", "summaries": "price is \ 
    reasonable"}] }}' -H 'Content-Type: application/json' \
    -X POST localhost:5002/invocations
```

# Ejercicios

-   Despliega el modelo usando el artefacto proporcionado por Azure ML
    en tu máquina local

-   Despliega el modelo en una imagen Docker

-   Despliega el modelo en una máquina Linux de Azure (Ver cómo crear
    este recurso y acceder a él
    [aquí](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal?tabs=ubuntu))

-   Despliega la imagen Docker a Azure ([Este
    recurso](https://docs.docker.com/cloud/aci-integration/) te puede
    ayudar)

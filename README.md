# Desarrollo-CI-CD

Desarrollo CI/CD
La Infraestructura como Código es un hecho en cualquier organización moderna. 

Desarrollar infraestructura declarativa nos garantiza que siempre la misma infraestructura va a ser deployada. 

Esta puede ser alterada (o no) por condiciones específicas al entorno, pero bajo un esquema controlado. 

Alterar el comportamiento basado en el entorno es muchas veces necesario debido a la naturaleza donde el software necesita ejecutarse (no existen los mismos requerimientos en un entorno de QA que en uno de Producción).



Nombre del Software:   Desarrollo de CI/CD
Versión:  TEAM/ONE/A.001.01
Fecha: 2022-09-14

------------

## Resumen del Software 👀

Al desarrollar una canalización de CI exitosa, las nuevas imágenes se envían al registro de Docker y las etiquetas de imagen respectivas se confirman en el repositorio de GitLab.

La acción de sincronización se activa a través de la acción de webhook de CI después de la confirmación o después de un intervalo de sondeo regular.

Argo CD se ejecuta como un controlador en el clúster de Kubernetes, que monitorea continuamente las aplicaciones en ejecución y compara el estado actual en vivo con el estado de destino deseado (como se especifica en el repositorio de GitOps).

El controlador detecta el OutOfSync del estado de la aplicación, lo que se hace al diferenciar las salidas de la plantilla de manifiesto de Helm y, opcionalmente, toma medidas correctivas.

Argo CD se integra a la perfección en los motores de Kubernetes y le permite adoptar la mentalidad de GitOps de inmediato.



------------


#imagen

--------------


## Requerimientos ☝️

Como requisitos obligatorios a tener para poder seguir la documentación, necesitaremos las siguientes tecnologías y conocimientos:

* Google Cloud Platform como plataforma Cloud (Se puede hacer con cualquier otro provider).
* Un clúster de Kubernetes (En nuestro caso, un GKE).
* Visual Studio Code o cualquier otro IDE en el cual escribir archivos YAML.
* Tener un mínimo conocimiento sobre objetos y recursos de Kubernetes (Para este caso, poder crear un Namespace, un Deployment, un Service y un Ingress/Load Balancer).
* Poder instalar ArgoCD a través de Helm como gestor de paquetes y artefactos.
* Entender cómo funciona kustomize.
* Saber utilizar Git.
* Tener una app creada con React y todas sus dependencias.
* Sumado al punto anterior, tener un repo creado con nuestro código para hacer el deployment (en este caso, el repo estará en GitLab).
* Saber cómo funciona Docker y poder crear un Dockerfile para poder construir y empaquetar nuestra app (se da por sentado que la persona que lee esto, ya tiene creado un repositorio en Docker Hub, si no es así, ir aquí).


Estos son los requisitos obligatorios para poder seguir la documentación. 

Particularmente nosotros hemos usado las siguientes dos tecnologías a modo de complementar:

* DonWeb para poder comprar y registrar un dominio en el cual se alojará nuestra aplicación.
* CloudFlare para poder agregar DNS personalizados a nuestra app y darle un extra de seguridad y profesionalismo.


---------------

# Lo que realizaremos 🐣

* Desarrollaremos una Infraestructura declarativa a través de manifiestos.
* Crearemos un CI/CD capaz de automatizar cualquier tipo de cambio realizado en nuestro código, creando así el mismo flujo en ambientes tanto de dev como prod, con la diferencia que los cambios a los manifiestos de dev serán automáticos y, los de prod, requerirán nuestra intervención manual.
* Desarrollaremos un CI sin nada que pueda tener credenciales o tokens hardcoded, sino que utilizaremos variables de entorno definidas en nuestra cuenta de GitLab, haciendo más seguro así nuestro CI.
* Utilizaremos Argo como CD para poder tener siempre un agente escuchando y comparando entre los manifiestos que se encuentran deployados y los manifiestos que están en nuestro repositorio.

--------------
#
#
#
# Paso 1
#
#
#

Utilizar Docker.

Composición del Dockerfile.

A continuación, veremos cómo está constituido el archivo de Dockerfile.



#reactapp-build

# Utilizar node12-alpine para hacer nuestro build
FROM node:12-alpine as build

# Especificamos dónde vivirá nuestra app
WORKDIR /app

# Copiamos toda la app de React al directorio
COPY . /app/

# Instalamos las dependencias necesarias para poder construir la app
RUN npm install
RUN npm install react-scripts@3.0.1 -g
# Construimos nuestra app
RUN npm run build

# Utilizaremos Nginx como servidor proxy para alojar nuestra app
FROM nginx:1.16.0-alpine

# Copiamos lo contruido en la primer parte del Dockerfile a la ruta declarada debajo
COPY --from=build /app/build /usr/share/nginx/html
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx/nginx.conf /etc/nginx/conf.d

# Exponemos el puerto 80 y lanzamos nuestro nginx
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

-------------------------
#
#
#
# Paso 2
#
#
#
--

Creación del CI.

A continuación, veremos todo el CI e iremos agregando comentarios parte por parte.


# Aqui empieza

-------------------

#Definimos los stages que queremos que se cumplan en nuestro CI

stages:
  - build-app
  - build
  - deploy-dev
  - deploy-prod

#Definimos variables (estas tienen un alcance o 'scope' global)

variables:
  IMAGE_NAME: "teamone-reactapp-repo"
  CI_DOCKERHUB_USER: "batimeunnescafe"

#Definimos nuestro primer stage, en el cual instalaremos node y las dependencias necesarias para que nuestra app hecha en React funcione


build-app:
  stage: build-app
  image: node
  script:
    - echo "Start building App"
    - npm install
    - npm run build
    - echo "Build successfully!"
  artifacts:
    expire_in: 1 hour
    paths:
      - build
      - node_modules/

# En este segundo stage, haremos el build de nuestra app basada en el Dockerfile escrito arriba. Usaremos un servicio de Docker "dind", que significa "Docker in Docker"
# Nótese que usamos muchas variables de entorno con la finalidad de hacer mucho más seguro nuestro código, ya que no exponemos ningún tipo de dato o credencial.

build:
  image: docker:19.03.12
  stage: build
  services:
    - docker:19.03.12-dind
    
  rules:
    - if: $CI_PIPELINE_SOURCE == "push"
    
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

    - docker build -t $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA .

    - docker tag $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:latest
    

    # PUSH IMAGE COMMIT SHA and LATEST

    - docker push $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:latest
    

#Aquí realizaremos nuestro primer deploy al ambiente "dev". Vemos que descargamos

#Kustomize y seteamos las configuraciones de Git. Luego, en el apartado "script"

#vemos que con Kustomize podemos editar los valores del manifiesto.

#También aclarar que este build tendrá como trigger un push hacia "master"

deploy-dev:
  stage: deploy-dev
  image: alpine:3.11
  before_script:
    - apk add --no-cache git curl bash
    - curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
    - mv kustomize /usr/local/bin/
    - git remote set-url origin https://${CI_USERNAME}:${CI_PUSH_TOKEN}@gitlab.com/andres.bmth.bt/The-One-DevOps-BootCamp.git
    - git config --global user.email "andres.bmth.bt@gmail.com"
    - git config --global user.name "batimeunnescafe"
  script:
    - git checkout -B master
    - cd deployment/dev
    - kustomize edit set image $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - cat kustomization.yaml
    - git commit -am '[skip ci] DEV image update'
    - git push origin master
  only:
    - master


#En este caso repetiremos todo lo anterior, con tres pequeños cambios.

#Primer cambio que notamos, hacemos un reset --hard para eliminar cualquier cambio que no tengamos pusheado, para luego hacer el pull correctamente (que es el segundo cambio)

#Como último cambio importante, vemos que este deploy sólo se realizará con una intervención manual nuestra.

#Una aclaración importante, vemos que tanto en los stages "deploy-dev" y "deploy-prod" en los commit, tenemos una anotación llamada [skip ci], la cual nos sirve para que no se nos genere un nuevo pipeline al hacer el nuevo cambio.

deploy-prod:
  stage: deploy-prod
  image: alpine:3.11
  before_script:
    - apk add --no-cache git curl bash
    - curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
    - mv kustomize /usr/local/bin/
    - git remote set-url origin https://${CI_USERNAME}:${CI_PUSH_TOKEN}@gitlab.com/andres.bmth.bt/The-One-DevOps-BootCamp.git
    - git config --global user.email "andres.bmth.bt@gmail.com"
    - git config --global user.name "batimeunnescafe"
  script:
    - git checkout -B master
    - git reset --hard
    - git pull origin master
    - cd deployment/prod
    - kustomize edit set image $CI_REGISTRY/$CI_DOCKERHUB_USER/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - cat kustomization.yaml
    - git commit -am '[skip ci] PROD image update'
    - git push origin master
  only:
    - master
  when: manual
  
------------
  
--------------------------------
 
 
 
#
#
#  
# Paso 3
#
#
#


Instalar y Configurar ARGO-CD

* Para poder instalar ArgoCD en nuestro clúster, debemos ejecutar los siguientes dos comandos para crear el namespace e instalar Argo.
* El tercer comando será para convertir el servicio a un Load Balancer y así tener una IP pública con la cual acceder a la GUI.

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'


* El objetivo de ARGO-CD es escuchar constantemente el continuo delivery/continuos deployment y lo hace de una forma en que corre casi nativo (Se instala dentro del mismo) en un clúster de Kubernetes. Es decir, verifica los manifiestos de los archivos de despliegue.
* La función principal es escuchar constantemente los cambios generados a través de las dos apps (Dev/Prod), que lo hace a través de kustomization. Tiene variantes de configuración, en este caso desde la app Prod, se sincronice de forma manual y el de Dev de forma automática, como también se produzca un DELETE los objetos previos a la nueva actualización.
* La mayor funcionalidad de ARGO-CD es la sincronización automática de los cambios generados.



# imagen

# imagen


--------------------------

#
#
#
# Paso 4
#
#
#


# Pushear al repositorio

Para comenzar a hacer funcionar el pipeline de CI / CD que hemos montado, solo tendremos que realizar un cambio en la aplicación en nuestro repositorio local y pushearlo.

Los comandos enseñados a continuación serán importantes para cumplir con lo anterior mencionado.
  
* git checkout master Para trabajar en la rama máster
* git pull  Actualiza el repositorio local.
* git add .  Agrega todos los cambios hechos al próximo commit.
* git commit -m "descripción-del-commit"  Crea una snapshot del proyecto.
* git push  Envía los archivos de la snapshot anterior al repositorio remoto.


## Algo importante que debemos fijarnos al momento de hacer nuestro push, es que se nos asigna un hash a nuestro commit, pero luego debido a los cambios que necesita hacer Kustomize para el deploy, ese hash cambiará, como podemos ver en las siguientes imágenes:


Recordemos el hash que se generó en PROD image update, ya que lo veremos en la última parte de esta documentación.


-------------

#
#
#
# Paso 5
#
#
#



# Ver los cambios.

* Para poder ver los cambios que hemos realizado en el paso anterior, deberemos asegurarnos que nuestro CI/CD haya pasado todos los cambios necesarios.
* Para esto, iremos a nuestro proyecto en GitLab, abriremos la opción de CI/CD y deberíamos ver nuestro Pipeline de la siguiente manera:

* Nótese como cada check pasado correctamente, es un stage que hemos definido arriba.
* Desde build-app, build, deploy-dev y deploy-prod.
* También debemos prestar atención al hash que nos muestra GitLab en los pasos posteriores como ya se aclaró en el paso anterior.
* Ahora ingresaremos a ArgoCD (si aún no sabemos la ip y tampoco sabemos cómo encontrarla, dentro de nuestra terminal haremos un kubectl get svc -n argocd)



imagen


Vemos que, en el ambiente de prod, tenemos todo actualizado, sincronizado y en estado "healthy" y, también podemos observar que el hash que está sincronizado, es el que se nos generó con el stage "deploy-prod"




----------------


Demo CI/CD - YouTube

# VIDEO












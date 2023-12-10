---
title: Instalar awx-operator con ks3
date: 2023-12-03
categories: [awx]
tags: [Linux, kubernetes]
image:
  path: /assets/img/awx-ks3.png
  width: 500 #normal 800
  height: 500 #normal 500
  alt: awx-operator
---
# Instalar awx-operator

Una referencia para poner en marcha Ansible AWX sobre un entorno de Kubernetes

Testeado:
+ debian
+ ubuntu

Requerimiento instalar con apt:
- git
- curl
- sudo
- python3
- python3-pip

## Instalacion

Instalar ks3 
````bash
curl -sfL https://get.k3s.io | sh -
````

Validar estatus
````bash
kubectl get nodes
````

Instalar kustomize
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
```

>Verifica el link en [Binaries | SIG CLI (kubernetes.io)](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/)

Desplegar awx operator.

Crear instrucciones de kustomize para instalar el operador
````bash
nano kustomization.yaml
````

>Asegúrese de especificar una versión de lanzamiento, usaré 2.2.1 en este ejemplo:

````yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.2.1

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.2.1

# Specify a custom namespace in which to install AWX
namespace: awx
````

Instalar operador
````bash
kustomize build . | kubectl apply -f -
````

Espere a que el operador esté instalado y funcionando
````bash
kubectl get pods -n awx
````

Crear awx.yaml
````bash
nano awx.yaml
````

````yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  nodeport_port: 30080
````

Agregue axw.yaml a `kustomization.yaml`
````yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.2.1
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.2.1

# Specify a custom namespace in which to install AWX
namespace: awx
````

Iniciar el trabajo para instalar AWX
````bash
kustomize build . | kubectl apply -f -
````

Después de unos minutos, se implementará la nueva instancia de AWX. Puede consultar los registros del pod del operador para saber en qué punto se encuentra el proceso de instalación
````bash
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager --namespace awx
````

Obtener contraseña de admin
```bash
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" --namespace awx | base64 --decode
```
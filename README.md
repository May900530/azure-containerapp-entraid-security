# Azure Container Apps con Autenticación Entra ID y Seguridad

Implementación de una aplicación web en Azure Container Apps con autenticación mediante Entra ID, utilizando Azure Container Registry y Service Principals para la gestión de accesos.

## Descripción del Proyecto

Este proyecto demuestra el despliegue de una aplicación Node.js en Azure Container Apps, implementando autenticación basada en Entra ID (Azure Active Directory) y aplicando principios de seguridad mediante RBAC (Role-Based Access Control).

## Arquitectura

La solución implementa los siguientes componentes de Azure:

- **Azure Container Registry (ACR)**: Almacenamiento privado de imágenes Docker
- **Azure Container Apps**: Plataforma serverless para contenedores
- **Container Apps Environment**: Entorno de ejecución con Log Analytics
- **Entra ID App Registration**: Proveedor de identidad para autenticación
- **Service Principal**: Identidad de servicio con permisos RBAC

## Requisitos Previos

- Azure CLI instalado y configurado
- Cuenta de Azure con permisos de contribuidor
- Git para control de versiones
- Node.js 18 o superior (para desarrollo local)

## Tecnologías Utilizadas

- **Azure Container Apps** - Plataforma serverless para contenedores
- **Azure Container Registry** - Registro privado de imágenes Docker
- **Azure Entra ID** - Servicio de identidad y autenticación
- **Node.js 18 Alpine** - Runtime de la aplicación
- **Express.js** - Framework web para Node.js
- **Docker** - Containerización de la aplicación
- **Azure CLI** - Herramienta de línea de comandos
- **Log Analytics** - Monitoreo y diagnóstico

## Recursos de Azure

### Resource Group
```
Nombre: rg-demo-ca
Ubicación: East US
```

### Azure Container Registry
```
Nombre: corpacr1750
SKU: Basic
Login Server: corpacr1750.azurecr.io
```

### Container Apps Environment
```
Nombre: ca-environment
Ubicación: East US
Log Analytics: workspace-rgdemocafsAd
```

### Container App
```
Nombre: corp-webapp-app
Image: corpacr1750.azurecr.io/corp-webapp:v1
Puerto: 8080
CPU: 0.25 cores
Memoria: 0.5 Gi
Ingress: Externo
URL: https://corp-webapp-app.salmonrock-7037116f.eastus.azurecontainerapps.io/
```

## Configuración de Seguridad

### Service Principal

Se creó un Service Principal con los siguientes permisos:

```bash
Nombre: demo-ca-sp
App ID: ae9ef61f-0243-4536-8876-42c9ba52ba11
Tenant ID: 652dee89-d926-459b-a0b7-2913a72f226d
```

**Roles asignados:**
- AcrPull: Permiso para extraer imágenes del Container Registry
- Contributor: Permisos de gestión sobre el Resource Group

### Autenticación Entra ID

La aplicación está protegida mediante autenticación de Entra ID con la siguiente configuración:

- Tipo de cuenta: Single tenant
- Redirect URI: `/.auth/login/aad/callback`
- Acción para solicitudes no autenticadas: HTTP 302 Redirect
- Token de ID habilitado

## Estructura del Proyecto

```
.
├── Dockerfile              # Configuración de la imagen Docker
├── package.json            # Dependencias de Node.js
├── server.js               # Aplicación Express
├── .gitignore             # Archivos excluidos del repositorio
└── README.md              # Documentación del proyecto
```

## Despliegue

### 1. Crear el Resource Group

```bash
az group create --name rg-demo-ca --location eastus
```

### 2. Crear Azure Container Registry

```bash
az acr create --resource-group rg-demo-ca --name corpacr1750 --sku Basic
```

### 3. Crear Service Principal

```bash
ACR_ID=$(az acr show --name corpacr1750 --query id --output tsv)

az ad sp create-for-rbac \
  --name demo-ca-sp \
  --role acrpull \
  --scope $ACR_ID
```

Guardar el output generado, específicamente el `appId` y `password`.

### 4. Asignar Rol Contributor

```bash
RG_ID=$(az group show --name rg-demo-ca --query id --output tsv)
APP_ID="<service-principal-app-id>"

az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope $RG_ID
```

### 5. Construir y Subir la Imagen

```bash
az acr build --registry corpacr1750 --image corp-webapp:v1 .
```

### 6. Crear Container Apps Environment

```bash
az containerapp env create \
  --name ca-environment \
  --resource-group rg-demo-ca \
  --location eastus
```

### 7. Desplegar Container App

```bash
ACR_LOGIN=$(az acr show --name corpacr1750 --query loginServer --output tsv)
IMAGE="${ACR_LOGIN}/corp-webapp:v1"
CLIENT_SECRET="<service-principal-password>"

az containerapp create \
  --name corp-webapp-app \
  --resource-group rg-demo-ca \
  --environment ca-environment \
  --image $IMAGE \
  --target-port 8080 \
  --ingress external \
  --registry-server $ACR_LOGIN \
  --registry-username $APP_ID \
  --registry-password "$CLIENT_SECRET" \
  --cpu 0.25 --memory 0.5Gi
```

### 8. Configurar Autenticación

La autenticación se configuró mediante Azure Portal:

1. Navegar a Container Apps > corp-webapp-app
2. Seleccionar Authentication en el menú lateral
3. Add identity provider > Microsoft
4. Configurar App Registration con tipo "Create new"
5. Establecer restricción de acceso a "Require authentication"

## Verificación

### Probar la aplicación

```bash
curl https://corp-webapp-app.salmonrock-7037116f.eastus.azurecontainerapps.io/
```

### Verificar autenticación

```bash
az containerapp auth show \
  --name corp-webapp-app \
  --resource-group rg-demo-ca
```

### Ver logs de la aplicación

```bash
az containerapp logs show \
  --name corp-webapp-app \
  --resource-group rg-demo-ca \
  --follow
```

## Endpoints de la Aplicación

- `/` - Endpoint principal (requiere autenticación)
- `/health` - Health check (requiere autenticación)

## Limpieza de Recursos

Para eliminar todos los recursos creados:

```bash
az group delete --name rg-demo-ca --yes --no-wait
```

## Seguridad

- Service Principal con permisos mínimos necesarios
- Credenciales no incluidas en el repositorio
- Autenticación obligatoria para todos los endpoints
- Container Registry privado con acceso controlado
- Logs centralizados en Log Analytics Workspace
- RBAC implementado a nivel de Resource Group
- Secrets almacenados de forma segura en Container App
- HTTPS habilitado por defecto en ingress externo

## Autor

**Maisbeiby Ramon**
- LinkedIn: https://www.linkedin.com/in/maisbeibyramon
- GitHub: https://github.com/May900530
- Email: may900530@gmail.com

## Agradecimientos

- Documentación oficial de Azure Container Apps: https://learn.microsoft.com/azure/container-apps/
- Documentación de Microsoft Entra ID: https://learn.microsoft.com/entra/identity/
- Comunidad de Azure y DevOps Engineers
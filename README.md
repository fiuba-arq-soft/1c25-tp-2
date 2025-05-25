# Punto de partida para el TP 2 de Arquitectura de Software (75.73/TB054) del 1er cuatrimestre de 2025

> La fecha de entrega para el informe y el código será el __*jueves 19/06 a las 17:59hs*__.
> La misma herramienta de GitHub Classroom nos va a mostrar el último commit que hayan hecho sobre `main` hasta ese momento, con lo que es un deadline fijo y estricto. :warning: :bangbang:

## Contexto

Para este Trabajo, migraremos el servicio de cambio de monedas de arVault a la nube, desplegado sobre Azure. Tomaremos como base la implementación del TP 1 con las modificaciones que realizaron ustedes (las que hayan funcionado mejor según las pruebas que realizaron).

En cada caso, deberán analizar y explicar qué está ocurriendo según lo que visualizan. Si encuentran algún cuello de botella o limitación, deben proponer y probar alguna táctica superadora, siempre que tenga sentido dentro del caso analizado. Discutan las diferencias entre tener la aplicación orquestada con Docker Compose en un equipo local y desplegada en la nube.

Tanto para escalar horizontalmente como para agregar una instancia de Redis, cada grupo deberá modificar/agregar archivos de Terraform como sea necesario.

### Storage

Para este TP, el storage deberá implementarse en Redis (o en algún otro motor de base de datos que Azure soporte en el modelo *platform as a service*). Si lo hicieron de manera opcional en el TP 1, la implementación es similar, solo cambian los datos de conexión.

Para crear una instancia de Redis en Azure, mirar el [recurso azurerm_redis_cache de Terraform](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/redis_cache). Les recomendamos mirar los archivos de Terraform que les damos para ver cómo pueden utilizar un archivo de configuración que se modifique dinámicamente al crearse la infraestructura y tener disponible los datos de conexión al Redis.

### Consulta a API externa

Simularemos la existencia de una API externa de ARCA mediante el despliegue de un contenedor Docker (basado en una imagen provista por la cátedra) en el servicio correspondiente de Azure.

Para poder interactuar con esta API, deberán agregar al body del request que recibe el servicio de ustedes un campo "dni", de tipo texto, que recibirá el número de DNI del cliente que quiere realizar el cambio de monedas. Este número deberá ser enviado por el generador de carga. Los números válidos de DNI irán de 1000000 a 99999999.

La API de ARCA provee un único endpoint, `/validar`, que recibe mediante POST un JSON como el que sigue:

```json
  {
    "dni": "40101235"
  }
```

El servicio validará que el número de DNI no tenga restricciones fiscales para efecutar operaciones de compra-venta de moneda extranjera, y devolverá el resultado de la validación, junto con un código de autorización (solo si no hay restricciones). Ejemplo:

```json
  {
    "ok": true,
    "codigoAutorizacion": "434351b7-952a-4a6c-9197-6759477b528c"
  }
```

El resultado debe registrarse en el log junto con el código de autorización (de haber sido aprobado), además de devolver una respuesta apropiada al cliente.

Se les recomienda generar métricas propias sobre el comportamiento de este servicio para correlacionarlo con el comportamiento del servicio de arVault.

Junto con el enunciado les damos una aplicación de ejemplo, para que vean cómo invocar al servicio y, además, puedan verificar la primera vez que levantan la infraestructura que todo funciona. El servicio de ARCA escuchará en la URI que se configura automáticamente en `node/config.js`. Vean el archivo `arca.tf`.

Una vez que hayan visto el uso en la aplicación de ejemplo, pueden reemplazarla por el código de ustedes. Traten de aprovechar el mecanismo de actualización de la configuración que les damos.

### Escalamiento

Probarán el funcionamiento del servicio inicialmente con 1 nodo y escalando a luego a 3.

Para escalar en un Virtual Machine Scaling Set (VMSS), deben ajustar el parámetro *instances* según lo deseado.

Para la administración de las instancias del VMSS se utilizará un [jumpbox](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/architectures/connect-to-environments-privately)

## Setup

### Archivo de variables

- Crear dentro de este repositorio un archivo `terraform.tfvars`:

    > __ATENCIÓN: EVITEN COMMITEAR ESTE ARCHIVO AL REPOSITORIO. LAS API KEYS EN GENERAL NO DEBEN COMMITEARSE. DE HACERLO DEBEN ELIMINARLAS Y CREAR NUEVAS__
    _

### Azure

- Crear una cuenta en [Azure](https://azure.microsoft.com/).
- Desde SSH Keys, generar un par de claves (key/secret). Dejar la clave privada en el directorio raíz del TP (o referenciarla con un link simbólico). Llamarla `key.pem`.
- Instalar [`Azure CLI`](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
- Ejecutar `az login` e iniciar sesión en Azure. Pegar la __subscription ID__ (que se muestra luego del login exitoso) en una variable llamada __azure_sid__ en `terraform.tfvars`, debe quedar de esta manera:

```properties
azure_sid = "<Subscription ID de Azure>"
```

### Datadog

- Crear una cuenta con el [pack estudiantil de GitHub](https://education.github.com/pack) en [Datadog](https://www.datadoghq.com/)
- Ir a `<Usuario> -> Organization Settings -> API Keys` y obtener la API Key.
  - En `terraform.tfvars`, asignar el valor de la API key a una variable llamda __datadog_key__, debe quedar de esta manera:

```properties
datadog_key = "<API key de Datadog>"
```

- Verificar el host en la URL del navegador:
  - Si es __app.datadoghq.com__, saltear este paso.
  - Si en vez de "app" aparece otro nombre (por ejemplo, __us5.datadoghq.com__) ver las secciones de "Configuración adicional" más abajo.

### Terraform

- Instalar [Terraform](https://developer.hashicorp.com/terraform) utilizando el package manager de la distribución de Linux que se utilice (recomendado), o descargándolo desde [aquí](https://developer.hashicorp.com/terraform/install).
- Revisar el archivo `variables.tf` y actualizar los valores default de las variables que corresponda. Para "location" (region), recomendamos utilizar "eastus2" o "westus2". Este archivo sí será commiteado, así que solo poner aquí valores default que puedan exponerse (para los demás, deben estar las variables declaradas aquí pero los valores deben estar en `terraform.tfvars`, que nunca hay que commitearlo).
- Ejecutar `terraform init`. Esto inicializa la configuración que requiere terraform, e instala los providers necesarios.

### Ansible

- Instalar [Ansible](https://docs.ansible.com/), ver [aquí](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) información para el SO que se tenga.
- Agregar el rol de Datadog para Ansible ejecutando: `ansible-galaxy install datadog.datadog`

#### Configuración adicional

Si en la verificación de Datadog encuentran que el host comienza con un string distinto a "app", deben editar el archivo `ansible/vmss_setup.yml` y colocar el hostname en la variable __datadog_site__.

## Correr los servidores

> __IMPORTANTE:__ En el script se utiliza el binario de `terraform`, asumiendo que se encuentra agregado a la variable `$PATH` (si lo instalan con un package manager, ya estará en el path).
>

Existe el script `start.sh` en la raíz del proyecto para crear la infraestructura y correr los servidores correspondientes.

```bash
# Inicializo terraform
terraform init
# Seteo los permisos apropiados para que la key sea válida
chmod 400 key.pem
# Guardo la clave pública que obtuve de Azure
echo <PUBLIC_KEY> > pubkey

# Script que inicia sesión en Azure, crea la infraestructura, la inicializa y provisiona la app
./start.sh
```

Terraform crea un archivo local llamado `terraform.tfstate` que tiene el resultado de la aplicación del plan. Usa ese archivo luego para detectar diferencias y definir un nuevo plan si hay modificaciones a la infraestructura. Ese archivo __no debe perderse__, pero como [puede contener información sensible en texto plano](https://www.terraform.io/docs/state/sensitive-data.html) no es recomendable commitearlo sin tomar algunas precauciones. Además, si se destruye y regenera la infraestructura, cambiará mucho, con lo que es muy propenso a conflictos en git.

La recomendación, por lo tanto, es que cada cual tenga su propia cuenta de Azure y de Datadog, y mantenga su propio `terraform.tfstate` en su computadora sin necesidad de compartirlo. [Acá](https://www.terraform.io/docs/state/remote.html) tienen más información e instrucciones sobre qué hacer si quieren operar todos los integrantes del grupo sobre una misma cuenta de Azure y compartir su tfstate.

### Verificación

Una vez levantados los servidores, se puede verificar su correcto funcionamiento utilizando la URL que se encuentra dentro del archivo `lb_dns` de la carpeta `node` y enviando un request:

```sh
curl `cat node/lb_dns`
```

Con esto pueden verificar que la aplicación Node de ejemplo levantó correctamente.

### Cambio de cantidad de instancias en el VMSS de la aplicación Node

Si se cambia la cantidad de instancias en el VMSS, una vez aplicados los cambios en la infraestructura, pueden ejecutar:

```sh
ansible/setup.sh
ansible/deploy.sh
```

Estos scripts actualizan el inventario con las direcciones IP de los nodos del VMSS y luego instalan la aplicación Node en cada uno. Puede ejecutarse todas las veces que sea necesario (si hay nodos que ya tienen las herramientas necesarias, para éstos la ejecución será más rápida).

### Re-deploy de la aplicación Node por modificaciones en el código

Si sólo se modificó la app Node y se la quiere actualizar (sin haber cambiado la cantidad de instancias), pueden ejecutar el script de deploy únicamente:

```sh
ansible/deploy.sh
```

### Administración de la aplicación Node

El proyecto usa [pm2](https://pm2.keymetrics.io/) para daemonizar y administrar la aplicación Node. Pueden ver [aquí](https://pm2.keymetrics.io/docs/usage/quick-start/) la documentación para acceder a los logs, detener y arrancar la aplicación, etc.

Si necesitan ingresar a alguna instancia de VMSS para ver logs o administrarla en general, deben ejecutar el siguiente comando desde la raíz del repositorio (reemplazando en el lugar apropiado por la IP interna de la instancia):

```sh
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ProxyCommand="ssh -o StrictHostKeyChecking=no -W %h:%p -q azureuser@$(cat ansible/jumpbox_dns) -i ./key.pem" -i ./key.pem azureuser@<IP privada del host>
```

Esto usa al Jumpbox de proxy SSH para evitar exponer con una IP pública a los nodos del VMSS. Utilizamos el mismo par de claves para el Jumpbox y los nodos por simplicidad. Normalmente se usan pares distintos.

### Envío de métricas a Datadog

El agente de Datadog se instala en cada instancia de Node automáticamente, cuando ejecutan el script `ansible/setup.sh` (el script `start.sh` lo invoca cuando crean la infraestructura).

Para enviar sus propias métricas, vean cómo hacerlo [aquí](https://docs.datadoghq.com/integrations/node/). Es muy sencillo porque el agente de Datadog abre el puerto de statsd en localhost y se encarga de enviar las métricas custom que recibe.

Para enviar métricas desde Artillery, vean la configuración [aquí](https://artillery.io/docs/guides/plugins/plugin-publish-metrics.html). Revisen el archivo `perf/run.sh` para colocar la API key en la variable `DATADOG_API_KEY`.

#### Configuración adicional

Si en la verificación de Datadog encuentran que el host comienza con un string distinto a "app", deben editar el archivo YAML de Artillery y colocar el hostname en la variable __apiHost__.

## Destruir la infraestructura

Cuando hayan terminado con el TP, pueden destruir la infraestructura con `terraform destroy`. Al par de claves que crearon a través de la UI de Azure deben eliminarlo también desde la UI.

## Cheatsheet de Terraform

```sh
# Ver lista de comandos
terraform help

# Ver ayuda de un comando específico, como por ejemplo qué parámetros/opciones acepta
terraform <COMMAND> --help

# Ver la versión de terraform instalada
terraform version

# Inicializar terraform en el directorio. Esto instala los providers e inicializa archivos de terraform
terraform init

# Ver el plan de ejecución pero sin realizar ninguna acción sobre la infraestructura (no lo aplica)
terraform plan

# Aplicar los cambios de infraestructura. Requiere aprobación manual, a menos que se especifique la opción `-auto-approve`
terraform apply

# Destruir toda la infraestructura. Requiere aprobación manual, a menos que se especifique la opción `-force`
terraform destroy

# Verifica que la sintaxis y la semántica de los archivos sea válida
terraform validate

# Lista los providers instalados.
terraform providers
```

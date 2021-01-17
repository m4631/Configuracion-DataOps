# Guía de Configuracion para DataOps

El objetivo de la presente es describir los pasos ejecutados para la configuración en Azure DevOps de los procesos de CI/CD correspondientes a proyectos de datos, promoviendo así prácticas de automatización para el despliegue íntegro, rápido y replicable de los diferentes ambientes de un proyecto: Desarrollo, Calidad y Producción.  

# Azure DevOps

## Configuración de organización y proyecto

Azure DevOps es una solución para equipos de tecnología que proporciona capacidades de planificación, colaboración y creación o implementación de aplicaciones. Extendiendo dichas capacidades a desarrollos tradicionales o implementaciones de otros proyectos, como en el actual caso donde nos referimos a la implementación de un proyecto de aprovechamiento de datos en la nube de Azure. 

Para iniciar a trabajar con Azure DevOps el primer paso es crear una organización. 

Dentro de esta organización pueden existir uno o múltiples proyectos. Crear o seleccionar uno existente para efectos de esta configuración.

Dentro de cada proyecto se encontrarán los siguientes componentes:
-	Azure Repos
o	Proporciona repositorios de Git o Team Foundation Version Control (TFVC) para el control de código fuente de su aplicación
-	Azure Pipelines 
o	Proporciona servicios de creación y lanzamiento para admitir la integración y entrega continuas de sus aplicaciones
-	Azure Boards 
o	Ofrece un conjunto de herramientas ágiles para respaldar la planificación y el seguimiento del trabajo, los defectos de código y los problemas mediante los métodos Kanban y Scrum
-	Azure Test Plans
o	Proporciona varias herramientas para probar sus aplicaciones, incluidas pruebas manuales / exploratorias y pruebas continuas
-	Azure Artifacts
o	Permite a los equipos compartir paquetes como Maven, npm, NuGet y más de fuentes públicas y privadas e integrar el uso compartido de paquetes en sus canalizaciones de CI / CD

Para efectos de la presente configuración, fueron utilizados Azure Repos y Azure Pipelines. 

# Azure Repos

## Ambiente de Origen

En algunos casos, partir de un ambiente existente, puede ser un buen punto de partida. Típicamente estos servicios originalmente son creados de forma manual y customizados hasta conseguir las características deseadas por el equipo de trabajo, los cuales se utilizan para la configuración en DevOps y su posterior replicación. 

En este documento, se toman como base una estrategia de este tipo.

## Configuración de código fuente

Dentro del actual proyecto, la sección de Azure Repos es utilizada para almacenar las plantillas de JSON (ARM Templates) que describen todos los componentes necesarios para la creación y configuración de los servicios en Azure. Estas plantillas son el corazón del proceso de replicación, pues en ellas se contienen todos los elementos que necesita Azure para ejecutar la creación de un servicio con todas sus características sin necesidad de intervención manual. Incluyendo incluso las configuraciones que se realizan dentro del servicio propiamente. Por lo tanto, estas plantillas en nuestro ejercicio es lo que llamamos “código fuente”.

Dentro de un repositorio de Azure DevOps, pueden existir múltiples ramas o Branches, representando diferentes líneas de tiempo del código fuente o diferentes componentes de este. Debajo la estructura de ramas del proyecto actual:
![image.png](/images/image-3c730ad2-1834-46dd-b84f-de4426fda4be.png)

Tomar en cuenta que esta estructura puede variar, pues los mismos son creados automáticamente según los cambios que se ejecuten en la interfaz visual de su servicio de Data Factory de origen.

A continuación, una descripción del propósito de cada rama:
-	master. Es la rama que representa los cambios aceptados y publicados en el data factory de origen.
-	adf_publish. Es la rama que contiene los ARM Templates que describen las características de construcción del servicio.
-	Otras ramas. Es un tipo de rama que se pueden crear por persona o funcionalidad donde se contemplan los cambios realizados específicamente por esa persona o para esa funcionalidad, sin afectar así el ambiente general que se define en master. Según se interactúe en el portal de Data Factory, más ramas de este tipo pueden ser creadas.

La rama principal para efectos del proceso de DataOps es adf_publish, pues como expresado anteriormente allí se encontrarán nuestras plantillas JSON.  Dentro de esta rama, la estructura de carpetas y archivos es la siguiente:
![image.png](/images/image-c469a3d9-226c-42f5-b1ac-b2d3938e3e07.png)

-	bi
o	Es un folder creado automáticamente por la integración de Azure DevOps con Data Factory que se explica debajo. Aquí se contienen las plantillas de creación del servicio de Data Factory.
-	OtherResources
o	Es un folder creado manualmente como parte de las configuraciones para almacenar la plantilla de creación del resto de recursos (excluyendo el Data Factory) que se encontraban en el Resource Group de origen.
-	Scripts
o	Es un folder creado manualmente como parte de las configuraciones para almacenar un script powershell que se utiliza en el proceso de despliegue del servicio de Data Factory. El script utilizado se encuentra documentado aquí https://docs.microsoft.com/en-us/azure/data-factory/continuous-integration-deployment#script

# Azure Pipelines

## Configuración de pipelines

En el componente de Azure Pipelines se configuró un Release Pipeline para la orquestación y automatización del despliegue hacia Azure en sus diferentes ambientes: Dev, QA y PROD.
![image.png](/images/image-b7213de2-a61b-4cf5-a9aa-96ea4d4eb87c.png)
 
Dentro de cada ambiente se describe el mismo conjunto de pasos. Según se muestra en la imagen:
![image.png](/images/image-f646a0c9-1aef-4e9a-b06a-e165d2fcd3f2.png)

-	ARM Template Deployment. 
o	Despliega el Data Factory por primera vez.
-	Azure PowerShell script
o	Detiene todos los detonadores que se encuentran configurados en el Data Factory de origen.
-	ARM Template Deployment
o	Despliega dentro del Data Factory del paso 1, los pipelines, datasets, linked services, integration runtimes, entre otros objetos. 
-	Azure PowerShell script
o	Reactiva todos los detonadores que se encuentran configurados en el Data Factory de origen.
-	ARM Template Deployment
o	Despliega el resto de los recursos que se encuentran en el Resource Group de origen.
-	Azure Powershell script
o	Recrea los containers o carpetas dentro del recurso de Data Lake.

## Variables en el pipeline

Para asegurarnos que en cada ambiente se desplieguen los recursos correctamente se definen variables con los valores relevantes para cada ambiente.  
![image.png](/images/image-7f398a63-c153-4e70-ab6c-467314479737.png)

Los secretos o valores sensitivos están siendo ocultados con la opción de “candado” que aparece al lado de cada valor. A la derecha se seleccionará el ambiente al que corresponde cada variable y su valor.

Tomar en cuenta que la variable “adminLoginPass” define las credenciales del usuario administrador con el que podrán iniciar sesión en el servicio de SQL. Favor contemplar su configuración y ocultación según lo hayan definido.

## Configuración de ambiente

Tomar en cuenta que para el ambiente de PROD se necesita conectar la subscripción de Azure correspondiente y definir el Resource Group (existente en la subscripción) que Azure DevOps estará usando para los fines de despliegue. 

### Seleccionar subscripción correspondiente
![image.png](/images/image-d7dc30ca-821f-40fe-a770-856e5972ba55.png)
 
Seleccionar cada uno de los pasos definidos en el listado de la izquierda (ver imagen) y modificar los valores de “Azure Resource Manager connection” y “Subscription” para apuntar a la subscripción correspondiente. Tomar en cuenta que en algunos pasos pudiera solo ser necesario completar el valor de “Subscription” mientras que otros además de Subscription también requerirán completar el campo de “Azure Resource Manager Connection”.

### Integration Runtime 

El data factory de origen contiene la representación original del Integration Runtime software que se encuentra instalado en una de las máquinas virtuales de su red. Esta representación original dentro del data factory de origen se recomienda mantener para efectos de mitigar los cambios a nivel de red que sean requeridos a raíz del proceso de instalación de un nuevo agente de Integration Runtime.

Tanto dev como qa pudieran utilizar esta referencia compartida que se encuentra en el data factory de origen. Para el caso de producción por el hecho de que usualmente se encuentra en otra subscripción y la sensibilidad propia del ambiente, se recomienda tener un Integration Runtime directo en este. 

Este cambio requiere una intervención manual para extraer del Integration Runtime que se crea en Azure dentro del Data Factory el valor de Key que necesitará para la instalación del software en su máquina virtual. 

### Variable DL Key

Ejecute un release del ambiente de Prod una primera vez. Esta primera vez pudiera resultar fallida a raíz de la falta de valor en una de las variables:
-	Variable DL_Key. Esta variable se encuentra vacía y tendrá que completarla con el valor del Access Key que puede encontrar en el servicio de Data Lake (a pesar del pipeline fallido, los recursos se han creado correctamente) que la primera ejecución habrá generado en el Resource Group correspondiente. 

# Documentación

-	Integración y despliegue continuo en Data Factory https://docs.microsoft.com/en-us/azure/data-factory/continuous-integration-deployment 
-	Integrar ARM Template en Azure Pipeline https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/add-template-to-azure-pipelines
-	Tarea de Powershell en Azure Pipeline https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/powershell?view=azure-devops 
-	Exportar ARM Template desde un Resource Group en el portal de Azure https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/export-template-portal 

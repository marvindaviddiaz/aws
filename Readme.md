# Conceptos de AWS

## CodeCommit
- **CodeCommit Notifications** para notificaciones básicas y **CodeCommit Trigger** para Invocar Lambdas desde CodeCommit
- La política que deben tener los developers es **AWSCodeCommitPowerUser**, y restringir que suban código a master con política Deny

## CodeBuild (buildspec.yml)
- En la definición se elige la imagen que hará el Build: Managed (no incluyen frameworks) o Custom, recursos (VPC), etc.. Lo mínimo son 3GB y 2 vCPUs.
- El archivo de configuración se compone de: Variables, Fases, Artefactos. 
- En el buildspec.yml se configura el **runtime-version** ejemplo: `docker: 18`, `nodejs: latest`.

## CodeDeploy (appspec.yml)
- CodeDeploy permite desplegar en **EC2/Onpremise (con agente CodeDeploy), Lambda, ECS**
- **Deployments Groups**: Son grupos de instancias ó ASG donde se harán los Deploys, **Si son instancias  se tiene que elegir por medio de Tags**
- **Hooks**: Pasos de cada etapa del Deploy: **ApplicationStop, DownloadBundle, BeforeInstall, Install,ApplicationStart, ValidateService**. Varían según el tipo de aplicación y el tipo de Despliegue: BlueGreen, In-place, etc.. 

### Deployment Configurations: 
- Donde se define el tipo de Deployment, **para EC2** tenemos los siguientes:
  	- **In-Place**:
		- AllAtOnce
		- OneAtATime
		- HalfAtATime
		- Custom (Ejemplo: Mínimo que el 80% siempre esté disponible al ir desplegando)
	- **Blue/Green**:
		- Se usan en conjunto con Auto Scaling Group ó con Instancias Fijas pero se deben crear antes de hacer el deployment
		- En BlueGreen el LB es necesario
- Para **Lambda y ECS** existen los siguientes:
	- AllAtOnce, 
	- Canary: **TWO INCREMENTS** Ej. Mandar el 10% del tráfico por 10 minutos, y si todo va bien, el 100% del tráfico pasa a la nueva versión
	- Linear: **EQUALS INCREMENTS** Ej. Mandar incrementos de 10% del tráfico cada 10 minutos (se completaría en 100 minutos ó 10 veces) 

https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html

### Rollbacks
- Los rollbacks se pueden hacer manuales ó automáticos, los automáticos pueden ser: 
 1. basados en que si Falla el Deploy
 2. Basados en Umbrales de alguna alarma de Cloudwatch (Ej, si el cpu de una nueva instalación sobrepasa el 75% hacer rollback)

### On premise Instances:
- Para desplegar OnPremise hay dos formas: 1. Creando IAM User y configurar con aws cli (forma sencilla, ambiente pequeños), 2. Con Rol y obteniendo credenciales temporales de STS usando un daemon (más segura y trabajosa).

## Code Pipeline:
- Por cada branch se debe crear un pipeline
- Los Artifacts de cada etapa se almacenan en S3 y son la entrada para la siguiente etapa, son diferentes a los Artifacts de CodeBuild
- Para iniciar el pipeline se puede de 2 formas:  Automático (CloudWatch Events) ó Periódico (CodePipeline Poll)
- Se pueden agregar **ManualAprovals** dentro de un paso del Pipeline para intervención humana (Deploy to Prod en un solo pipeline)
- Cuando un pipeline se define por CF, el atributo `runOrder` en los StageAction indica el órden de ejecución.
- Se puede ejecutar un lambda desde el pipeline. Con `PutJobSuccessResult`, `PutJobFailureResult` se manda el resultado. Ej. Hacer un Request 200 al sitio web después del deploy, 
- Un buen CU para CodePipeline es desplegar templates de CloudFormation, también con CF se pueden crear pipelines 

## Code Star:
- Es como CodePipeline pero con un UI más sencillo y con plantillas por lenguaje de programación y/ó frameworks. Tiene dashboardspara ver los builds, monitoreo, jira, etc. En las opciones del proyecto se puede ver toda la configuración asociada: buckets, roles de servicio, CF.. 
- Por debajo usa CF, CodeCommit, CodeBuild, CodePipeline, según la plantilla que se usó.
- A bajo nivel CodeStar está configurado con un CF template.yml usando una transformación `AWS::CodeStar`

## Jenkins:
- Jenkis puede reemplazar CodeBuild / CodeDeploy / CodePipeline y/ó trabajar con alguno de estos, pero esta solución no es serverless
- **EC2 Plugin**: Lanza EC2 para usar como Workers.
- **CodeBuild Plugin**: Usa CodeBuild en lugar de agentes fijos (~serverless).
- **ECS Plugin**: Usa ECS en lugar de agentes fijos (~serverless).

## Code Artifact
- Para almacenar dependencias: Maven, Gradle, npm, yarn, pip.
- Tambien funciona como proxy para los repositorios centrales almacenando una copia de los artefactos.
- **Upsteam Repositories**: Descargar dependencias de un solo endpoint y el Upsteam apuntar hasta a 10 repositorios.
- Los Domains sirven para agrupar repositorios, lo ideal es que en una sola cuenta este CodeArtifact y por medio de dominios dar acceso a otras cuentas de AWS.

## Code Guru
- Automatiza las revisiones de código, usa ML. **CodeGuru Reviewer** (analisis estático) y **CodeGuru Profiler** (performance en ejecución)
- Soporta Java y Python, se integra con Github, Bitbucket, CodeCommit. Puede ser usado en aplicaciones corriendo en AWS o OnPremise
- **CodeGuru Reviewer Secrets Detector** Usa ML para detectar secrets, passwords, llaves, etc. en el código
- Cuando se asocia CodeGuru Reviewer con un repositorio de CodeCommit, **automáticamente analyza los pull requests.**


## Elastic Beanstalk
- En EB se puedeb crear ambientes de 2 tipos: Web Server, Worker (Tareas de larga duración ó calendarizadas).
- Los tipo **Workers** crea 2 colas en SQS para procesar trabajos: `WorkerQueue` y `WorkerDeadLetterQueue`, adicionalmente crea un archivo cron.yaml, para ejecutar tareas calendarizadas.
- Cuando se unsa EB Cli los valores especificados en el comando tiene prioridad a los archivos .ebextensions
- La BD debe crearse como parte del ambiente de Beanstalk ó por aparte, dependiendo si se quiere que la BD forme parte del mismo ciclo de vida de la aplicación de Beanstalk, ya que si queremos conservar la BD aunque se elimine el ambiente lo mejor es crear la BD por separado.

- **Deployments**
	1. **All at once**: Es la forma más rápida, pero hay downtime 
	2. **Rolling**: Pocas instancias a la vez (bucket), luego el siguiente bucket. No hay downtime, la capacidad se reduce mientras se despliega. 2 versiones coexisten durante el despliegue.
	3. **Rolling With Additional Batches**: Como la anterior solo que se crean nuevas instancias para desplegar el batch, permitiendo operar a la capacidad máxima. 2 versiones coexisten. Al final de actualizar las instancias las que se crearon se borran, bueno para producción, es un poquito más caro que Rolling
	4. **Inmutable**: Levanta nuevas instancias en un nuevo ASG, desplegar la versión en esas nuevas instancias, luego mete esas instancias al ASG original y finaliza la versión vieja. No hay downtime, Es la opción más cara, En caso de fallas es la más rápida de hacer rollback (solo se finaliza el nuevo ASG), bueno para producción
	5. **Blue/Green**: No es un feature de Beanstalk, es más trabajo manual: Crea un nuevo ambiente "Stage" y despliega la nueva versión ahí, la nueva versión (green) puede validarse y hacer rollback si no cumple, Luego con Route 53 (Weighted Policies) se le manda un porcentaje pequeño de carga Ej 10%, y al final usando Beanstalk usar la opción "Swap URL" para intercambiar URLs entre ambientes

## EC2 Image Builder
- Automatizar la creación, mantenimientoy validación de AMIs o Container Images, puede publicar AMIs a multiples regiones y cuentas.
- Con **AWS RAM** (Resource Access Manager) se pueden compartir Images, Recipes, Components a otras cuentas
- Se recomienda almacenar el AMI ID en SSM Parameter Store, para que los templates de CF tengan siempre la última versión de las AMIs

## ECS
- Las instancias corren el agente de ECS y usan una AMI especial para ECS, el agente es un contenedor de docker
- ECS crea automáticamente un ASG para subir la cantidad de nodos
- **ECS Task Definition**: Es metadata en formato JSON que le dice a ECS como correr un Docker Container, No está amarrada al Clúster
- **TIP**: Si un Task no puede descargar la imagen de ECR, no puede hablar con S3 revisar el `Task Role` que tiene asociado el Task
- **TIP**: Se puede usar CodeDeploy para hacer Blue/Green Deployments en ECS
- **ECS Service**
	- Ayuda a definir cuantas tareas deben correr y como deben correr, pueden linkearse a ELB / NLB / ALB
	- **Tipo de Servicio**: REPLICA: se indican cuantos se necesitan, DAEMON: uno en cada nodo
	- **Tipo de Deployment**: 
		Rolling Update: Default manejado por `ECS`
		Blue/green: Manejado por `CodeDeploy`
	- **Task Placement**: Para customizar en qué instancias del cluster van a levantarse las tareas, por defecto se usa `AZ Balanced Spread`
	- Para rutear el tráfico de múltiples contenedores y que usan puertos random, ALB usa `Dynamic Port Forwarding` 
	- No se puede asignar un LB a un ECS Service creado, solo en la creación. Adicional, El LB se crea primero en EC2 y luego se usa en ECS

### Fargate
- Para escalar solo se incrementa el numero de Task, Simple. Sin aprovisionar servidores
- No soporta HostPort como al desplegar en EC2.
- También soporta los Deployments: Rolling Update y Blue/Green

### ECS + Beanstalk
- Se puede correr Elastic Beanstalk en modo Single y Multi Docker Container 
- Multi Docker ayuda a correr multiples containers por EC2 en EB
- Por debajo crea: ECS Cluster, EC2 Instances, ASG, LB, Task Defintions
- Requiere un Dockerrun.aws.json en los fuentes, y la imagen debe ser pre subida en ECR

### Roles en ECS 
- **Classic**: A las instancias se les asigna un Rol el cuál tiene los permisos para modificar el clúster, descargar imagenes de ECR, etc. Cuando se lanzan Services el Service tiene un TaskDefinition y ese a a su vez tiene un TaskRole, ese TaskRole tiene un TrusRelationship para que solo pueda ser asumido por ECS y es el que va a tener los permisos que necesitan los contenedores /aplicación. 
- **Fargate**: Cuando se crea un clúster ECS Fargate, como no hay instancias solo se crea el Rol TaskRole. (Más simple)

### Autoscaling
- En ECS se configura a nivel del Servicio, minimo, máximo, Scaling Policy, etc..
- Hay 2 políticas de autoescalamiento:
	- **Target Tracking**: Se monitorea en base a una métrica Ej: CPU y en base a eso se incrementan los Task
	- **Step Scaling**: Basada en una Alarma (CPU, Memoria, etc..)  y en base a eso se incrementan los Task
- Hay un problema con el autoescalamiento si el clúster es Classic, ya que lo que se incrementa son los Task, y no las instancias. 
- El escalamiento de ElasticBeansTalk con ECS, si incrementa las instancias y los Task, en fargate no se tiene este problema ya que teoricamente no hay instancias.

### Cloudwatch Integration
- A nivel de Contenedor se puede definir la configuración de Log, se le puede indicar el driver (awslogs, splunk), si es awslogs se pueden enviar directo a Cloudwatch Logs.
- Si se usan instancias de EC2 entonces se debe instalar el Agente de Clouddatch y enviar los archivos del SO a Cloudwatch Logs por ejemplo `/var/log/docker` 
- A nivel del cluster de ECS tambien se puede activar el `Cloudwatch Container Insights` para colectar Métricas (CPU, memoria, Disk, etc.. A NIVEL DE CONTENEDOR y enviarlos a Cloudwatch, para utilizarlo en las métricas, dashboard, etc.. (Tiene un costo adicional)
- Con Cloudwatch Events se pueden configurar reglas basadas en los eventos de ECS ej: cuando se termina un container, quiere levantar otra tarea, enviar una alarma, ejecutar un comando, etc..

## Cloudformation:
- En CF hay una opción que lleva al Calculator para estimar el costo mensual basado en un template
- Para hacer referencias entre stacks se usan los `Outputs e !ImportValue` respectivamente
- Nunca actualizar un Nested Stack directamente, siempre hay que actualizar el Stack Padre.
- Para borrar un Stack que tiene un bucket primero se tiene que borrar el contenido del bucket
- Para desplegar Lambdas en cloudformation, se puede subir el código a S3 y además especificar la versión del Zip de S3 (`S3ObjectVersion`) esto con el fin de que al cambiar el tag versión CF desplegará los cambios ya que el template cambiará
- El log de un script UserData se almacena en `/var/log/cloud-init-output.log`
- `Fn:Base64` sirve para codificar un string en base64 ideal para el UserData de una EC2
- `!FindInMap [MyRegionMap, !Ref "AWS::Region", 32]`

- **cfn-init**: 
	- Correr scripts de una manera más elegante, EC2 se comunica con CF para bajar los scripts en formato yaml
	- Los script se crean mediante: `AWS::Cloudformation::Init` y permiten definir (packages, files, commands, sevices, etc.. como Ansible)
	- Para llamar cf-init se hace desde el UserData:
		`/opt/aws/bien/cfn-init -s ${AWS::StackId} -r MyInstance --region ${AWS::Region}`
	- Los logs de cfn-init se almacenan en `/var/log/cfn-init.log`
- **cfn-signal**: 
	- Para que el template se quede esperando el resultado de cfn-init se usa **`cfn-signal`**
		`/opt/aws/bien/cfn-signal  -e $? SampleWaitCondition ...`
- **cfn-hup**: 
	- Levanta un Daemon en EC2 que se queda escuchando si hay nuevos cambios en la Metadata de EC2
		`/opt/aws/bin/cfn-hup ...`
	- `/etc/cfn/cfn-hup.conf` almacena el nombre del stack y las credenciales para el Daemon 
	- El Intervalo de chequeo para cambios en la metadata es de 15 minutos, pero se puede cambiar
	- Ejemplo: Verificar cada 2 minutos si la metadata `CloudFormation::Init` ha tenido cambios, y si ha tenido ejecutar un script
- **cfn-get-metadata**: Script para obtener la metadata
- **ChangeSet**: Sirve para ver los cambios que se harán a un Stack, luego se pueden aplicar. Para estar más seguro.
- **Deletion Policy**: `Retain`, `Snapshot` Backup antes de borrar, `Delete` default para todos los recursos (Excepto RDS::DBCluster que es Snapshot)
- **Termination Protection**: Para prevenir que eliminen el Stack.
- **DependsOn**: Este recurso debe crearse hasta que X recurso se cree primero
- **Parameters from SSM**
	- Los parámetros del Stack pueden ser de SSM: `Type: AWS::SSM::Parameter::Value<String>`
	- Si se actualiza el stack usando la misma plantilla y los parámetros de SSM cambiaron, se actualizan los recursos afectados por el cambio de parámetros.
	- Hay parámetros públicos de AWS que se pueden usar en los template:
		`'/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'`
- **Custom Resources**:
	- `Custom::MyCustomResource` hacen referencia a funciones Lambda. que son invocadas cuando ocurren eventos en el Stack: CREATION, DELETE_IN_PROGRESS, ETC. 
	- CU: Crear un Custom Resource que hace referencia a un lambda que limpia un bucket cuando el Stack entraba en DELETE_IN_PROGRESS.
- **Drift Detection**:
	Ayuda a detectar si los recursos creados con el Template aún están igual (IN_SYNC) o no (DRIFTED), muestra las diferencias si no lo estan en una tabla y en formato JSON
- **UPDATE_ROLLBACK_FAILED**: https://repost.aws/knowledge-center/cloudformation-update-rollback-failed
- **InsufficientCapabilitiesException**: Cuando no se han puesto los capabilities necesarios.
- **Stack Policies**:
	- Parecidas a las políticas de IAM donde podemos restringir acciones sobre recursos específicos del Stack, estas políticas se validan cuando se actualiza el Stack
	- Aunque se puede modificar la política temporalmente para permitir acciones específicas, posiblemente por modificaciones urgentes, pero luego de aplicar la modificación, la política regresa a su estado original con el que fué creada.
- **Cloudformations StackSets**
	- Para desplegar Stacks en múltiples cuentas / regiones
	- Una vez desplegado el StackSet en múltiples regiones ó cuentas se pueden agregar más regiones después.
	- CUs
		- Habilitar AWS Config en todas las regiones ó cuentas donde esté trabajando.
		- Habilitar CloudTrail en otra cuenta 
		- Habilitar GuardDuty en otra cuenta

## AWS CDK - CLOUD DEVELOPMENT KIT
- Sirve para definir infraestructura usando lenguajes de programación: JS, TS, Java y .Net (para no usar Yaml/Json), Genera CF por debajo
- Utiliza componentes de alto nivel llamados **Constructs** `const vpc = new ec2.Vpc(this, "MyVpc", {...` 
- La ventaja de usar código es que los errores aparecen en el IDE, a diferencia de CF Yaml que hay que subir y esperar a que falle

## LAMBDA
- El tiempo máximo de ejecución es 15 minutos. Si una tarea se tarda más de eso lo mejor sería usar AWS batch
- Para funciones asincronas: Se puede configurar un DLQ para cuando Lambda falla más de 3 veces
- Por defecto son 1000 lambas por 'Reserve Concurrency', se puede incrementar.
- Por temas de compliance se puede activar el log de las invocaciones en CloudTrail
- Aparte de Api Gateway tambien se puede integrar directo con ALB (pero se pierden los features de de api gateway, ej. autenticacón)
- Lambda por si mismo integra KMS para tener variables encriptadas en tránsito, y en la función se debe hacer el decrypt en ejecución, y en código ya no es necesario indicar la llave criptográfica porque ya se configuró a nivel de lambda
	`boto3.client('kms').decrypt((CipheredtextBlob=b64decode(os.environment["DB_USER"])))`
- **Cross-Account EFS Mounting**: Para que un Lambda en otra cuenta pueda montar un EFS es necesario establecer un VPC Peering y que la política del EFS permita `ClientMount` y `ClientWrite` a la otra cuenta (parte `Principal`) 
- Una versión en un lambda engloba el código y la configuración, nada se puede cambiar (Versión = INMUTABLE)
- Alias es un puntero a una versión (Alias = MUTABLE)
- A nivel de Alias se puede usar una segunda versión adicional con un % de tráfico asignado.
	CANARY = TWO STEPS (ej: primero 10%, luego 100%)
### SAM
- Permite correr la fución local: `sam local invoke "HelloWordFunction" - e events/event.json`, Levantar el API Local: `sam local start-api` 
- SAM se integra con CodeDeploy para poder hacer Canary Deployments, es necesario agregar unas líneas a la configuración de la función. Al hacerlo se crea una aplicación en CodeDeploy usando Alias, permite hacer rollback y configurar Alarmas, también permite llamar funciones Hooks para validación del deployment.

### Step Functions
- Para **Orquestar Flujos** incluyendo intervención humana, pueden durar hasta 1 año.
- Step Function también se puede usar en conjunto con AWS BATCH para orquestar los Jobs cuando son tareas muy largas.
- En la consola se puede ir viendo las tranciciones en el flujo, el input y output de cada paso, etc..

## API GATEWAY
- El tipo de EndPoint puede ser **Regional** (Toda la región), **Edge Optimized** (Menor latencia usando todas las Edge Locations),  **Private** (en la VPC usando un VPC Endpoint)
- Api Gateway se puede integrar con Lambda, cualquier endpoint HTTP, Mock, AWS Service, VPC Link
- Los Stage se pueden usar para manejar 2 versiones simultaneamente /v1, /v2. Se puede hacer rollback de un Stage ya que el historial se almacena
- Las Stage Variables son como variables de ambiente pero para API Gateway, pueden ser usados en Lambda ARN, HTTP Endpoints, Parameter Mapping Templates, también son pasadas a la función Lambda a travéz del objeto `context`. Se usan comunmente para indicar a que Alias de las funciones lambda llamar.
- A nivel de Stage se pueden habilitar Canary, y asignarle un porcentaje del tráfico a los nuevos deployment del Stage, luego después de la aceptación se hace Promote al Canary y todo el tráfico se mueve a esa versión del deployment.
- API Gateway tiene un límite de 10k RPS a nivel de CUENTA (Se puede incrementar), `429 Too many Requests`
- Con los **Usage Plans** se puede limitar el número de RPS para un API Key. Se puede limitar por Segundo, Por Mes. Se puede granularizar a nivel de API + Stage + URL + Method.
- **Tip**: Es posible tener Throttles a nivel de API Gateway, a nivel de Usage Plan y a nivel de Lambda.
- Desde API Gateway se puede invocar a un StepFunction de forma asíncrona. solo hay que mandar la estructura del json requerido por StepFunction.
- Se puede definir el API usando **Open API** para que AWS valide los request antes de enviarlos a los lambda. Con `x-amazon-apigateway-request-validators` se le puede indicar si queremos validar todo, ó solo el body o solo los parámetros

## OpsWorks
- OpsWorks consta de 3 partes: OpsWorks Stack, OpsWorks for Chef Automate, OpsWorks for Puppet Enterprise
- Es para las personas que usan Chef OnPremise y luego quieren migrar a la nube y seguir usando Chef
- Un Stack es un grupo de Layers, Instaces and AWS Resources.
- Los tipos de Layers que se pueden crear son: **Opsworks, ECS, RDS**
- **Auto Healing**: Si el agente de OpsWork no se puede comunicar con AWS en 5 minutos, la instancia es reiniciada. Con CloudwatchEvents podemos definir una regla para ser notificados.
- Tipos de instancias:
	- **24x7**: Instancia Normal
	- **Time**: Permite definir horarios para levantar y bajar las instancias en base semanal.
	- **Load**: Estas se levantan o bajan automaticamente en base a la carga (CPU, memoria, etc..) Como Autoescale pero no tan flexible
- **Apps**: Aplicaciones que se encuentran en repositorios GIT o Http y contienen los Cheef Cookbook
- Se pueden crear Deployments en las instancias utilizando las Apps.
- **TIP**: AWS OpsWorks Stack Lifecycle Events:
   Cada Layer tiene un set de 5 Lifecycle Events, cada uno tiene asociado un set de Recipes. Los eventos son:
   
   1. Setup    : Cuando la instancia ha terminado de bootear
   2. Configure: Cuando la instancia entra o sale "En Línea", Asocia o Desasocia una IP a una instancia, Attach o Detach un ELB a un Layer.
   3. Deploy   : Cuando se despliega la aplicación a un sert de instancias.    
   4. Undeploy : Cuando se hace undeploy o se borra la aplicación de un grupo de instancias
   5. Shutdown : Antes de Terminate la instancia EC2
   De los 5 eventos "Configure" es el único que cuando ocurre afecta a TODAS las instancias del Stack.

## CloudTrail
- El trail **puede** capturar los eventos de todas la regiones
- CloudTrail envía los archivos a S3 (SSE-S3) pero también se puede configurar para enviarlos a CloudWatch Logs (Ambos al mismo tiempo)
- Estructura del Log: `userIdentity`, `eventTime`, `eventSource`, `eventName`, `requestParameters`, `responseElements`
- Puede haber un Delay de hasta 15 minutos por parte de CloudTrail, **para eventos en tiempo Real se recomienda usar Cloudwatch Events**
- El digest de los archivos aparece cada hora
- **Integridad de los logs**: `aws cloudtrail validate-logs --trail-arn ... --start-time ... `
- Para mandar los archivos de Cloudtrail a un S3 de otra cuenta (CrossAccount), hay que modificar el BucketPolicy para permitir escrituras y lecturas de otras cuentas.
- **CloudTrail supports much more events than the events that are sent by AWS services to EventBridge.**
- **EventBridge Integration**:
	X SERVICIO HACE ALGUNA ACCIÓN  ==>    CLOUDTRAIL   ==>  EVENT BRIDGE
- https://tusharf5.com/posts/aws-eventbridge/

## SQS
- Reintentos: Después de que **MaximumReceives** es excedido el mensaje se va a un DLQ. La DLQ se debe crear manualmente, y asociarla a la cola principal.
- El DLQ de una FIFO debe ser también FIFO, El DLQ de un Standard también debe ser Standard
- Se recomienda tener la retención a 14 días en la DLQ, para temas de DEBUGGING
- Cuando el código se corrije se debe **Redrive** los mensajes de la DLQ a la Cola normal
- **A nivel de SNS también se pueden enviar los mensajes fallidos a SQS DLQ**

## EventBridge
- Permite archivar los eventos (todos/filtrados) para luego poder replicar escenarios fallidos.
- Permite poder recibir todos los eventos de una organización a una sola cuenta.
- Los eventos tambien se pueden enviar a Cloudwatch Logs, y se les puede aplicar **Transformations** para modificar el mensaje usando templates.
- Adicionalmente tiene eventos de Otros Partners (Symantec, Sugar CRM, Datadog...)
- **CloudTrail Integration**: 
	Para trabajar con eventos más específicos, se debe seleccionar el Servicio Ej: EC2 y en el tipo de evento se debe seleccionar `AWS API Call via CloudTrail` y acá ya podemos usar eventos más específicos como por ej: `CreateImage` (Las operaciones List, Get, Describe no son soportadas por EventBridge). En el ejemplo se creaba una alarma cuando crean un AMI
- **S3 Integration**:
	- Forma 1: Dentro de S3 en Opciones Avanzadas está la opción de Events, podemos elegir que eventos serán los disparadores (PUT, COPY, Delete, ect..) y luego "Send To" que permite llamar a SNS, SQS, Lambda. En esta forma no todos los eventos de S3 están soportados.
	- Forma 2: Hacerlo en EventBridge, pero primero hay que habilitar que CloudTrail grabe los eventos de Data de S3 a nivel de todos los Bucket o solo de alguno es específico. Esto para tener el soporte de Eventos a nivel de Objects de S3

## Cloudwatch
- La información de las métricas depente del intervalo de tiempo que tengan por ej: Métricas que se reportan cada minuto la data se almacena por 3 horas, y la que se reporta cada hora está disponible 15 meses.
- **RAM, Espacio en Disco, y Número de procesos,** son métricas que solo se obtienen al usar Custom Metrics ó el Agente de Cloudwatch
- Métricas que se obtienen con el agente: cpu, disk, space, io, ram, network, processes, swap
- A nivel ASG se pueden activar "Group Metrics" para ver métricas de EC2 agrupadas por ASG
- En las categorías de las métricas ej: "EC2" se puede dar a la opción "Automatic Dashbaord" crea un dashboard las métricas importantes, las métricas se pueden exportar.
- Metrics **Standard Resolution**: 1 minuto de granularidad. **High Resolution**: 1 segundo de granularidad
	`aws cloudwatch put-metric-data --metric-name FunnyMetric --namespace Custom --value 123 --dimensions InstanceId=1123,...`
- **TIP**: How to Correlate Data? "Cloudwatch Dashboard"
- Se pueden obtener metricas y logs de Instancias Onpremise con el CloudWatch Agent
### Cloudwatch Alarms
  - Las acciones de una alarma pueden ser Notificaciones (SNS), AutoScaling, EC2 ACtions, System Manager Action
  - Una Cloudwatch Alarm no puede ser un evento de entrada para un CLoudwatch Events
  - En us-east-1 se puede crear una Billing Alarm, basado en el total estimado ó por servicio
### Cloudwatch Logs
  - Para configurar la Retención de los Logs se hace a nivel de LogGroup
  - El agente tiene un wizard y forma un json que puede ser almacenado en SSM ParameterStore para que las instancias lo bajen de ahí.
  - **Metric Filter**, sirve para buscar coincidencias en Cloudwatch Logs. De un MetricFilter se puede crear un Metric y del Metric un MetricAlarm
  - Se puede exportar data de Cloudwatch Logs a S3, se selecciona el Log Group y se puede filtrar por fechas ó Stream, y el bucket puede ser CrossAccount.
	Se puede automatizar desde Cloudwatch Events, creando una Regla Periódica y llamando a un lambda
  - Por medio de la opción de "Subscription" se pueden analizar los logs en tiempo real usando:
  	 - Kinesis Stream, Firehose
  	 - Lambda
  	 - OpenSearch

  - **AWS Managed Logs**:
    - Cloudtrail Logs => S3 y CWL
    - VPC Flow Logs =>  S3 y CWL
    - Route 53 Access Logs =>  CWL
    - Load Balancer Access Logs (ALB, NLB, CLB) => S3
    - S3 Access Logs => S3
    - Cloudfront Access Logs -> S3

### Synthetics Canary
  - Scripts configurables que monitorean APIs, URLs, Websites..
  - Reproducir lo que los clientes hacen en el sitio antes que los clientes sean impactados.
  - Los scripts se escriben en Nodejs o Python
  - Se usa Headless Google Chrome

### Anomaly Detection
  - Monitorea y analiza las métricas para determinar anomalias usando algoritmos de ML

## Amazon Lookout for Metrics
- Detecta anomalias en las metricas e identifica la causa raiz usando ML
- Es mucho más completo que CW Anomaly Detection

## EC2 Instance Status Checks

- Checks automáticos que identifican problemas de hardware y software.
- **System Status Checks**: Problemas con AWS, perdidas de energía, problemas físicos. Resolución: Parar  e iniciar la instancia
- **Instance Status Checks**: Monitorea software y configuraciones de Red en la instancia. Resolución: Reinicia la instancia o cambia la configuración de la misma.
- Desde EC2 se puede crear una Alarma para notificar cuando estos checks fallan. Además se puede configurar de forma automática si queremos hacer el reboot o recover.

## X-Ray
- **Keywords** para el examen: Traces, Debbuging, Distributed application
- Se puede automatizar con Lambda la detección de latencia, errores, etc, en una aplicación.
- Tambien se puede integrar con Elastic Beanstalk (de forma visual o en el archivo, además se debe verificar que el rol de la aplicación tenga acceso a XRay)
	options_settings:
	  aws:elasticbeanstalk:xray:
	    XRayEnabled: true`
- Hay un producto llamado **AWS Distro OpenTelemetry** que es como XRay solo que OpenSource. Pueden enviar a Xray, Prometheus, Third Party Solutions..
- El CU para usar OpenTelemetry es si queremos estandarizar con OpenSource la Telemetría o si queremos enviar los Traces a multiples destinos simultáneamente.

## Athena
- **Servicio Serverless de Queries** que permite analizar data almacenada en S3 en SQL
- Soporta CSV, JSON, ORC, Avro, Parquet
- $5 por TB scaneado
- Comunmente usado con Quicksight para reportes/dashboard
- CU: **BI / Analytics / Reportes, Hacer queries a VPC Flow Logs, ELB Logs, Cloudtrail trails**, etc..
- Performance:
  - Para mejorar el costo se recomienda usar **columnar data** (menos scans). Para esto se debe usar el formato Apache Parquet o ORC. Usar Glue para convertir la data a Parquet o ORC
  - **Comprimir la data** usar bzip2, gzip,..
  - **Particionar** los datasets in S3 para buscar la data específica leyendo menos. Usar el nombre de la columna y valor para el directorio
  	`s3://bucket/path_table
	 s3://bucket/path_table/year=2023
	 s3://bucket/path_table/year=2023/month=1
	 ...
	`
  - **Usar archivos mayores** a 128MB para minimizar overhead	
 - **Federated Query** Permite configurar un Lambda para obtener información de donde sea (ElastcCache, DocumentDB, RDS, Dynamo, Redshift, DB Onpremise) parsearla y almacenarla en S3 para que Athena la use.

## SSM
- Ayuda a administrar EC2 y sistemas On-Premise a escala, detectar problemas, parcheo. Funciona para Windows y Linux
- Servicio gratuito integrado con CloudWatch, AWS Config
- El agente viene por defecto instalado en las Linux AMI y Ubuntu AMI
- **TIP**: si una instancia no se puede controlar desde SSM, puede ser un problema con el agente ó con el Rol 

- **Default Host Management Configuration (DHMC)**:
	- Cuando se activa en FleetManager, todas las instancias que se lancen en esa región se vuelven **Managed Instances** manejadas por SSM (Session Manager, Patch Manager, Inventory, Fleet manager.
	- Es necesaio que el agente esté actualizado a las últimas versiones.

- **Hybrid Activation**: 
	- Opción en SSM donde se pueden crear credenciales para usar el agente de SSM en máquinas OnPremise, estas credenciales tienen asociado un ROL que es el que usa el agente. 
		`sudo amazon-ssm-agent -register -code .. -id .. -region`, luego `systemctl start amazon-ssm-agent` 
	- Una vez que se ha registrado la instancia OnPremise aparece en "Managed Instances" en SSM y se le pueden asociar Tags.

- **Resource Group**: Sirve para agrupar recursos en base a etiquetas ó en base a los recursos de un template de CF

- **Run Command**: Permite correr "Commands Documents" (Documentos en formato Json) 
	- Cada comando tiene sus parámetros de entrada. Y se puede ejecutar seleccionando instancias específicas, usando Tags ó por medio 
	de ResourceGroups
		1. Command
		2. Automation
		3. Policy
		4. Session	
	- Cuando se corre un comando se puede especificar el número ó porcentaje de instancias que correran el comando simultaneamente
	- También se puede parar en base a un número o porcentaje de errores
	- Los tipos de documentos son: 
	- También se puede sacar la salida de los comandos a S3 ó Cloudwatch, también enviar SNS Notificaciones acerca del estado.

- **Patch Manager**
	- Se pueden crear custom "Patch Baseline" donde se pueden aceptar y rechazar un listado de parches
	- Patch Sources, sirve para indicar la fuente de los parches.
	- Para aplicar un Patch BaseLine, primero se crea un Maintenance Windows.
	- El Maintenance Window sirve para especifícar cuando queremos parchear las instancias.
	- Luego a la MaintenanceWindows se registran los "Targets" (instancias ya sea específicas, por tags, ó resource groups)
	- Luego a la MaintenanceWindows se le asocia un Run Command: `AWS-RunPatchBaseline`
	- Después ya solo queda esperar que se ejecute el comando según la periodicidad indicada
	- Una vez que las instancias son parcheadas en la opción de 'Compliance' se pueden ver las instancias en verde

- **Inventory**
	- Sirve para recolectar metadata de las instancias Instancias y OnPremise.
		Aplicaciones, AWS Components, Files, Configuración de Red, Windows Updates, Services, Instance Details, etc.. 
	- El Top de aplicaciones instaladas
	- Corre cada 30 minutos

- **Automations**
	- Automatiza la administración de instancias y recursos
	- Funciona también con Documentos (json)
	- Al ejecutar un Automation, pide los parámetros de entrada del documento y se puede visualizar como se ejecutan los pasos
	- Tareas communes:
		- **AWS-StopInstaceWithApproval** para solicitar aprobación de 2 usuarios para detener una instancia
		- **AWS-StopInstace** para parar una instancia a una hora específica
		- **AWSSUpport-ExecuteEc2Rescue** para recuperar instancias que se volvieron inalcanzables por temas de red, firewall, etc..
		- Simplificar la creación de Golden AMIs
	- Automation está integrado a Cloudwatch Events. Se pueden configurar alarmas o acciones cuando un Automation finaliza (error o éxito)

- **Sesion Manager**
	- Se guardan todos los comandos ejecutados en la sesión ya sea en S3 ó Clouwatch Log
	- Se establece desde el navegador (Sin llaves)
	- El usuario que usa es ssm-user

- **SSM AppConfig**
	- Permite Configurar, Validar y Desplegar configuraciones dinámicas a las aplicaciones.
	- No se necesita reinicar la aplicación
	- La configuración se puede almacenar en: Parameter Store / SSM Documents / S3


## AWS Config
- Ayuda a la auditoría y grabación a travéz del tiempo del compliance de los recursos de AWS, envía alertas, integración con EventBridge
- **No previenen que los cambios pasen, pero si se pueden hacer Remediations**
- Se puede almancer la configuración en S3 para analyzar con Athena
- Se pueden usar reglas predefinidas o escribir las propias usando Lambda
- **Configuration Recorder** almacena la configuración de los recursos a travéz del tiempo (se crea automáticamente cuando se habilita Config)
- Para evitar que desabiliten Config se recomienda usar una SCP a nivel de la organización
- **Aggregators** centralizan la data de config en todas las cuentas
- **Conformance Packs** Colección de **Config Rules** y **Remediations** empaquetadas en Yaml (similiar a CF) para desplegar en la organización, acá más que todo se configuran las reglas existentes de AWS ya con sus respectivos parámetros de entrada. Ej: Para la regla `iam-password-policy` van a tener una longitud de 14 como mínimo. **Enfocadas a cuentas Individuales y Organización**
- **Organizacional Rules** Son Reglas de Config que aplican a todas las cuentas de una organización, similar a los Conformance packs, solo que estás están **enfocadas a una organización**.
- Util para dar seguimiento a la configuración de todos los recursos de una cuenta
- Cada regla que se agrega tiene un costo de $1 al mes
- Las reglas se pueden validar siempre que hayan cambios o de forma periódica
- También tiene Remediation Actions
- **CustomRules**: 
	- Se configuran usando Lambda
	- Pueden ser invocadas por cambios en la configuración o de forma periódica
	- Puede monitorear cambios en los tipos de recursos asociados, Tags, Todos los cambios
- Por cada regla se puede definir un Remediation Action (Por debajo usa SSM Automation usando las de AWS o definiendo un Custom RemediationAction). Por cada RemediationAction se definen Retries, timeout, parámetros de entrada.
- Notificaciones que Config puede enviar a un SNS Topics. La idea es recibir notificaciones cuando:
  - Cambio en la configuración de un recurso
  - Cuando se reciben cambios de la configuración en nuestra cuenta
  - Compliance State de los recursos si cumplen con las reglas
  - Cuando inicia una evalución de una regla
  - TIP: Las notificaciones se parametrizan a nivel de todo Config, no a nivel de Reglas por ejemplo.

- También Cloudwatch Rules se integra con Config, por lo que se pueden parametrizar notificaciones y acciones en Cloudwatch Events tomando de base los eventos de Config. 

- **Agregation View**:
	- Un Agregator es un Recurso de AWS Config que colecta Configuraciones de Config y data de compliance de:
		 Múltiples cuentas y multiples regiones, 
		 Una cuentas multiples regiones, 
		 Una Organization y todas las cuentas de esa organización, 


## SERVICE CATALOG

- Service Catalog es simple y restringe muchas opciones a los usuarios
- Tareas del Admin: Crear Productos (CF Templates) -> Crear Portfolio (Colección de productos) -> Control IAM Permissions to Porfolios
- Tareas del Usuario: De un Product List -> Usar Productos Aprovisionados que ya se encuentran correctamente configurados.
- Ayuda a Governance, Compliance y Consistency
- Ayuda a los usuarios a lanzar productos (CF Stacks) sin tener mucho conocimiento
- Los Output de CF son super importantes porque es la forma en que los usuarios verán los endpoints creados, direcciones, etc..

## INSPECTOR

- Inspector busca vulnerabilidades por Instancia, Container y Accesibilidad de Red
- Para hacer un escaneo a una instancia, es necesario que SSM esté activado y las instancias tengan el Rol de 
	`AmazonSsmRoleForInstancesQuickSetup`


## EC2 COMPLIANCE

- Config
	- Se asegura que la instancia tenga una correcta configuración  (Que no tenga el puerto ssh abierto, etc..)
	- Audita el compliance a lo largo del tiempo

- Inspector
	- Vulnerabilidades de seguridad ejecutadas DESDE ADENTRO de la instancia usando el agente de inspector
	- O desde afuera usando networki scanning sin necesidad de usar el agente

- Systems Manager
	- Correr Automatizaciones, parches, comandos, inventario a escala

- Service Catalog
	- Restringir como las instancias pueden ser lanzadas minimizando las configuraciones
	- De mucha ayuda para los usuarios principiantes

- Configuration Management
	- SSM, Opsworks, Ansible, Chef, Pupper, User Data
	- Se asegura que las instancias tengan la configuración apropiada



## PERSONAL HEALTH DASHBOARD

- Se integra con Cloudwatch Events para notificar o realizar acciones en base a los eventos de Health de todos o algunos servicios que afectar a la cuenta.
- Cuando un AccessKey y SecreteKey son expuestas en un repositorio de Github público, AWS monitorea y envia un evento que puede ser Capturado en CloudwatchEvent y realizar una tarea automatizada. En el ejemplo se llama un StepFunction que elimina las credenciales, hace un query en Cloudtrail para ver que hicieron con esas credenciales, luego notifica por correo electrónico el resultado de cloudtrail.


## TRUSTED ADVISOR

- Da recomendaciones para la cuenta, tiene 2 tier, la segunda tiene más recomendaciones y es más cara. Se centra en:
	- **Cost Optimization**: Low Utilization EC2 Instances, Idle LB, Underutilized EBS VOlumnes, Unassociated IP Addreses, ...
	- **Performance**: Alta utilización de Instancias, EBS Volumnes..
	- **Security**: MFA en root, Security Groups, Public Snapshots, S3 Permissions, IAM Access Key Rotation 
	- **Faul Tolerance**: Chequea la edad de los Snapshots, Optimización de LB, Redundancia de Tunel de VPN, RDS Backups...
	- **Service Limits**: Chequea cuando el se llega al 80% de los límites de varios servicios
- Tambien se puede integrar con Cloudwatch Events, para detecta y reaccionar a los cambios de estado de los Checks de Trusted Advisor. CUs:
	- Enviar una notificación a Slack cuando hay un cambio de estado en el Check
	- Push Data sobre los Checks hacia Kinesis Stream para monitoreo en tiempo real.
	- Parar una instancia con baja utilización
	- Crear Snapshots de EBS que no tengan un backup reciente
	- Borrar IAM Keys expuestas y monitorear el uso (Mismo CU visto en Health)
	- Habilitar el versionamiento en S3 Bucket
	- (Algunos de estos CU también se pueden hacer sin Trusted Advisor)
- Para que aparezcan los Eventos en Cloudwatch hay que situarse en us-east-1 por ser un servicio global, el tipo de evento principal es `Chek Item Refresh Status`
- Cuando se paga soporte Business ó Entreprise,  en Cloudwatch aparecen más Metricas de Trusted Advisor
- Por defecto TrustedAdvisor se refresca varias veces al día, pero se puede automatizar y tener un proceso para que cada 5 minutos esté ejecutando el `refresh-trusted-advisor-check` y para consultar el `describe-trusted-advisor-check-result`

## GUARD DUTY
- Detecta amenazas de forma inteligente en la Cuenta usando Machine Learning
- Analiza CLoudTrail Logs, VPC Flow Logs, DNS Logs
- Notifica en cazo de algún hallazgo y se integra con AWS Lambda
- Puede detectar si una instancia está minando Bitcoin
- Detecta Ataques de fuerza bruta
- Tambien se integra con Cloudwatch Events para automatización, el evento es `GuardDuty Finding`. Es importante hacer estás integraciónes para ser notificados ó tomar accones en tiempo real.

## MACIE
- Ayuda a analizar data sensible en S3 y da insights acerda de ella.
- Analiza si en la data hay:
	Private Keys (DSA, EC, PGP, RSA) 
	Encrypted Data Blocks
	Password etc shadow
	Secret Keys (Facebook, Github, Slack, etc..)
- Tiene integración con cuentas externas
- Se puede customizar varias cosas (extensiones de archivos, expresiones regulares, etc..)

## SECRETS MANAGER
- La principal diferencia con Parameter Store, es que acá se pueden rotar los secrets y se integra con RDS.
- $0.40 por secret por mes

## LICENSE MANAGER
- Para manejar el licenciamiento de varios proveedores: Microsoft, Oracle, Sap, etc..
- Se pueden definir reglas para aumentar el número de licencias.
- Se pueden asociar AMIs a la reglas, así cuando se lancen instancias de esa AMI, se obtiene una licencia del Pool definido.

## COST ALLOCATION TAGS
- Son como los Tags, pero se muestran como columnas en los Reportes
- **User Tags**: Definidos por los usuarios (los tags normales), Inician con el Prefijo **user:**
- **AWS Allocation Tags**: Solo se muestran en el Billing Console, prefijo aws: Ej: `aws:createdBy`, NO son aplicadas a los recursos que se crearon antes de la activación
- **Cost Allocation Tags**: Solo se muestran en el Billing Console, NO son aplicadas a los recursos que se crearon antes de la activación. De los Tags Normales se seleccionan cuales van a ser también CotstAllocationTag

## TIPS

- PHI = Protected Health Information 
- PII = Personally-Identifying Information
- Direct Connect = Conexión privada directa entre site y AWS
- VPN = Site-to-Site para protejer a nivel de internet
- Network ACL: stateless firewall a nivel de VPC
- WAF: Seguridad web contra Exploits
- Security Group: Staful firewall a nivel del Hypervisor de instancia 
- System Firewall: Instalar firewall dentro de EC2 (Linux firewalls, Windows Firewall)

## ASG

- Un ASG puede ser lanzado desde un Launch Configuration (Opción antigua) ó desde Launch Template (Opción moderna)
- Launch Configuration no maneja versionamiento, Laun template Sí
- Un Launch Template es una forma mas avanzada de definir instancias. Desde un Launch Template podemos lanzar:
	- Instancias Individuales
	- AG
	- Spot Fleet
- Launch Template da la flexibilidad de lanzar diferentes tipos de instancia cuando se está creando el AG ej: t2.micro y large, así como lanzar unas OnDemand y otras Spot
- Cooldown = Tiempo que debe pasar antes de que otro evento de escalamiento suceda (scale in, scale out).

### Scheduled Actions: 
En está opción del AG se puede cambiar el min/max/deseado a una hora específica.

### Scaling Policy: 

 - **Target Tracking Scaling**:
	- En esta opción se basa en métricas (CPU, LB Request COunt, Network Bytes In/Out), y cuando se excede del porcentaje deseado de todo el AG, se aumenta el "desired" creando así una nueva instancia. 
	- Aca existe una opción llamada "disable scale-in" para que las instancias nuevas que se levantan no se borren incluso si la métrica ya bajó, de este modo el AG solo va a crear instancias no a bajarlas.
	- Cuando se crea un Scalin Policy también se crea un Alarm en Cloudwatch que es el que realmente controla el AG.

 - **Simple Scaling Policy**:
	- Acá primero hay que crea una alarma en Cloudwatch, luego seleccionar si queremos Subir/Bajar N Instancias.
	- Suena mas sencilla que la anterior, pero la ventaja que da es que como se baja en una Alarma propia, podría ser en respuesta de cualquier servicio de AWS.

 - **Scaling Policy With Steps**:
	- También trabaja con una alarma independente (como la anterior) pero permite definir múltiples pasos ejemplo:
		Agregar 	1 Instancia 	Cuando 		40  	<= 		CPU Utilization  	<= 70
		Agregar 	2 Instancia 	Cuando 		70  	<= 		CPU Utilization  	<= 90
		Agregar 	3 Instancia 	Cuando 		90  	<= 		CPU Utilization  	<= +infinity


### ALB Integration:
- Cuando se crea el ALB se tiene que crear un TargetGroup donde se le indica que va a ser de tipo Instance (también puede ser IP ó lambda). Luego se edita el ASG para asociarle el TargetGroup del ALB, así cuando el ASG escale, agregue o elimine esas instancias al TargetGroup del LB. (Según veo el TargetGroup del ALB es un concepto virtual de agrupación para hacer posible que las instancias de un ASG puedan ser asociadas a varios TG de diferentes LBs, El campo permite varios TargetGroup)
- A nivel del TargetGroup del LB se puede configurar "Slow start duration" para que una instancia empiece a recibir tráfico después de N segundos, para que una nueva instancia no empiece a recibir gran cantidad de tráfico desde el inicio.

### Suspending and Resuming Scaling Processes
 - A nivel de ASG se puede suspender varios tipos de eventos:
	Launch, Terminate, AddToLoadBalancer, AlarmNotification, AZRebalance, HealthCheck, InstanceRefresh, ReplaceUnhealthy, ScheduledActions
 - Si se suspende el `Launch` ó el `Terminate` si se modifica el "Desired", no se subirian o bajaran instancias respectivamente. Si por ejemplo se suspenda el HealthCheck, ya no se harían Healthchecks
 - Si se suspende `ReplaceUnhealthy` permitiría poder hacer Troubleshoting a instancias que estén teniendo problemas y que el ASG las esté terminando por Unhealthy.
 - Si se suspende el `AddToLoadBalancer` lo que hace es que agrega la instancia al ASG, pero no la agrega al TargetGroup del LB, para que no reciba tráfico. Después se puede asociar manualmente al TargetGroup para que reciba tráfico. (Hay que aprenderse todos estos eventos usando el Bookmark.)
 Adicional a los eventos anteriores, si queremos aislar una instancia del ASG para troubleshoting, se puede hacer de dos formas: 
 	1. `Detach`: La quita del ASG y crea una nueva
		2. `Set to Standby`: No la quita del ASG, pero el LB no le manda tráfico.
Otra opción importante es `Scale In Protection` a una instancia de un ASG se le puede activar y con esta opción el ASG nunca le va a dar Terminate.

### ASG Lyfecycle Hooks
- Procesos que se pueden meter para retrasar el cambio de estado de una instancia en el ASG. Normalmente el Flujo de una instacia es:
	Pending ----> In Service  ......  Terminating ----> Terminated
  Don están las flechas entran 2 estados más `Wait` y `Proceed`. Wait se queda esperando hasta que se le confirme que puede Proceder ó no.
	`aws autoscaling complete-lyfecycle-action ...`

- Se puede crear desde la consolta de ASG ó desde Cloudwatch Events

- CUs:
	- Instalar o configurar software cuando se lanza una instancia.
	- Guardar los logs en S3 antes de Terminar la instancia
	- Tomar un snapshot de EBS antes que la instancia se termine

### AGS Termination Policies
- Default: evalúa lo siguiente:
	1. Cual AZ tiene mas instancias, y busca la que no tenga scale in protection
	2. Mira el Allocation Strategy (On-Demand, Spot)
	3. Mira cual tiene el mas viejo LaunchTemplate/LaunchConfiguration
	4. por último si hay varias que hagan match, selecciona random.
- OldestInstance
- OldestLaunchConfiguration
- NewestInstance
- ClosestToNextInstanceHour
- AllocationStrategy
- OldestLaunchTemplate


**TIP**:
	Si tenemos una Instancia que procesa mensajes de la cola, y no queremos que por alguna razón sea Terminated mientras esté procesando
	antes de empezar a leer de la cola se puede activar Scale In Protection desde el sdk y cuando termine de leer se desactiva. No suena optimo, pero así lo hacen. 
	Este CU surgió porque en el ejemplo se aumenta o decrementa el Desired de un ASG basado en la cantidad de mensajes que hay en una cola.

### CreationPolicy & UpdatePolicy
- Si se está creando un ASG desde CloudFormation y queremos que el ASG falle si las instancias no se lanzan bien, se puedar configurar un 
	`CreationPolicy` en el ASG, en el que le decimos por ej: que queremos esperar N Signals desde las máquinas, y sino se reciben el ASG fallaría al crearse. Los signals se envían usando el `cfn-signal` (CF bootstrap scripts) que ya se había visto 
- Con la propiedad de CF `AutoScalingRollingUpdate`, podemos especificar cuantas instancias queremos que estén atendiendo y cuantas se van a actualizar en un ASG para poder hacer un RollingUpdate.
- Tambien existe la propiedad de CF `AutoScalingReplacingUpdate`, acá lo que hace es crear un nuevo ASG y cuando la CreationPolicy se valida, entonces se elimina el viejo ASG

## DEPLOYMENTS STRATEGIES (REVIEW)

- In Place 		(1 LB, 1 TG, 1 ASG)	(dowtime, las mismas instancias se actualizan)
- Rolling  		(1 LB, 1 TG, 1 ASG, nuevas instancias)
- Replace  		(1 LB, 1 TG, 2 ASG, nuevas instancias)
- Blue / Green  	(2 LB, 2 TG, 2 ASG, nuevas instancias, R53) (Acá manda el Route 53: Simple / Weighted (Mandar un poco de tráfico primero))

## DYNAMO
- Indices Secundarios Locales solo se pueden crear cuando se crea una tabla, después solo se pueden crearn secundarios globales. Además los Locales tiene que tener la misma Llave primaria que la tabla, solo el Sort cambia.
- La capacidad On-Demand (siempre va a funcionar bien) es mucho más cara que Provisioned. Cuando se selecciona Provisioned se definen los Read/Write Capacity Units ó el mínimo y máximo para escalamiento.
### DAX: 
- Clúster de Dynamo a nivel de tabla para lecturas intensivas, elegimos el tipo de instancia de los nodos y cuantos queremos, para que la data más accesada se mantenga en memoria.
- Sube el performance hasta 10x, de milisegundos a microsegundos, inclusive millones de peticiones por segundo.
### Dynamo Streams
- Para enviar los eventos de los items a un lambda, se puede configurar para enviar varios al mismo tiempo (batchs), se puede enviar solo las llaves, la nueva, vieja ó ambas versiones del item. Loguea INSERT, UPDATE, DELETE
- Con Dynamo Streams solo podemos conectar 2 lambdas al mismo tiempo, por el límite de Kinesis Streams que es lo que usa por debajo. Si quisieramos tener 3 lambdas conectadas, lo mejor sería tener un lambda conectada y ese Lambda haría el envío a SNS Topics y luego a este Topic si tener varias Lambdas conectadas.
### Global Tables
- Solo se puede habilitar cuando esta activo Dynamos Streams y la tabla esté vacía.
- Sive para tener una replica de la tabla en otra region
- Si se insertan en la replica también se reflejan en la principal (Two-Ways replication)
### TTL
- Tiempo de expiración de los Items
- Cuando se elimina el registro también se manda el evento a Dynamo Streams, y si está habilitado GlobalTables, también se borra de las replicadas.
### CUs
- Cuando escriben a S3, llamar a un Lambda para que en Dynamo almacene la Metadata de los archivos, para poder: buscar por fecha, calcular el espacio de cada usuario en el bucket, listar objetos con ciertos atributos, listar objetos que se subieron cierto rango de fechas.
- Construir un API para buscar Items con:  Dynamo + Dynamo Streams + Lambda + Elastic Search

## Multi AZ in AWS
- Servicios donde Multi-AZ debe ser habilitada manualmente:
	- EFS, ELB, ASG, Beanstalk: Asignar AZ
	- RDS, ElasticCache: Multi-AZ
	- Aurora:  Data es almacenada atravéz de multi-AZ. Puede tener multi-AZ para la BD (tal como RDS)
	- ElasticSearch:	Multi master
	- Jenkins: Multi Master
- Servicios donde MUlti-AZ es implícito:
	- S3: (Excepto OneZone-InfrequentAccess)
	- DynamoDB
	- Todos los Managed Services propietarios de AWS
- EBS
	- Está atado a una sola AZ. Que se podría hacer para que EBS fuera Multi AZ?
	1. Tener un ASG con 1 min/max/desired
	2. Tener un Lyfecycle hooks para Terminate: Que haga un snaphot del EBS volume
	3. Tener un Lyfecycle hooks para Start:	Que copie el snapshot, cree un EBS y lo atache a la instancia nueva

TIP: Sí un EBS está usando PIOPS, para obtener el máximo performance después del snapshot, se debe leer el volumen entero una vez (pre-warming of IO blocks)


## Multi Region Services
- DynamoDB Global Tables (Active-Active, Habilitado por Streams)
- AWS Config Aggregators (Multi Región y Multi Cuenta)
- RDS Cross Region Read Replicas (usado para lectura y disaster recovery)
- Aurora GLobal Database (una región master, la otra es para lectura y disaster recovery)
- EBS Volumes snapshots, AMI, RDS Snapshots (Pueden ser copiados a otras regiones)
- VPC peering permite el tráfico entre regiones
- Route53 usa una red global de servidores DNS
- s3 Cross Region Replication
- Cloudfront es un CDN Global
- Lambda@Edge para funciones Lambda Globales en las Edge Locations

**TIPS de Route 53**
- Los HealthCheks Automatizan DNS Failovers, Los HealthCheks pueden monitorear:
	1. Endpoint
	2. Otros HealthChecks (Calculated Health Checks)
	3. Cloudwatch Alarms (FULL CONTROL: throttles de Dynamo, Custom Metrics..)

Los HealthChecks también ser integran con CW Metrics para enviar alertas, etc..

## Multi Region CodePipeline
- El pipeline debe copiar los artefactos a las diferentes regiones para que así CodeDeploy en diferente región pueda encontrar los artefactos.

## DISASTER RECOVERY 

- **RPO**: Cada cuanto se corren backups
- **RTO**: Cuando tiempo dura el Downtime

Estrategias:
- **Backup and Restore**:  
	Fácil, no es muy caro, High RPO/RTO
- **Pilot Light**: 
	Una versión pequeña de la aplicación siempre está corriendo en la Nube. Más rápido que Backup/Restore. Pero se necesita un servicio de replicación de Data 			
- **Warm Standby**:
	Sistema Full arriba y corriendo, pero a la mínima cantidad, en desastres se escala la carga.
	Tambien se necesita servicio de Replicación de Data
- **Hot Site / Multi Site Approach**: 
	Todo el sistema corriendo en AWS y OnPremise
	Mínimo RPO/RTO
- **All AWS Multi Region**
	Todo el sistema corriendo en 2 regiones diferentes
	Aurora Global Master y Slave

Tips:
- **Backups**:	
EBS Snapshots, RDS automated backups
Desde OnPremise: Snowball o Storage Gateway
- **High Available**:
Route53, RDS, ElastiCache, Site to Site VPN, Direct Connect
- **Replication**:
RDS Replication, Aurora + Global Databases
Storage Gateway
-**Automation**:
CF, EB, Lambda
-**Chaos Testing**
Netflix "simian-army"

## On-Premise Strategies with AWS
 - Habilidad de descargar las Linux 2 AMI como VM (.iso format) (Vmware, Kvm, Virtualbox)
 - VM Import / Export
		Migrar aplicaciones existentes a EC2, para disaster recovery
 - AWS Application Discovery Service
		Para recolectar información sobre los servidores OnPremise y planear una migración
		Migracion Hub
 - AWS Database Migration Service (DMS)
		- On-Premise => AWS, 	AWS => AWS, 	AWS => OnPremise
 - AWS Server Migration Service (SMS)
		Replicacion Incremental de OnPremise a AWS (en ejecución)

## AWS ORGANIZATIONS 

- Es un servicio Global
- La cuenta principal es 'master' y las demas 'members', las members solo pueden pertener a una organización.
- El Billing se consolida a todas las cuentas.
- Se puede habilitar CloudTrail en todas las cuentas para enviar los logs a un S3 Central.
- Tambien enviar los Cloudwatch Logs a una cuenta central de Logging
- Las OU agrupan varias cuentas, pueden haber OU adentro de un OU
- Cuando se crean las cuentas se crea un rol en cada una `OrganizationAccountAccessRole` que es asumido por la master para realizar todo lo que se necesite.
- Por defecto los descuentos de Instancias Reservadas y Saving Plans son aplicados a toda la organización, hay que desabilitar que sean compartidos, para que solo apliquen a las cuentas que querramos.

### Service Control Policies - SCP

- Permitir o bloquear IAM actions
- Aplicadas a nivel de OU ó a nivel de Cuenta. Estas no aplican a la Master Account
- SCP son aplicadas a todos los Usuarios y Roles de la cuenta, incluido Root
- SCP no afectan a 'service-linked' roles (Estos roles ayudan a los servicios de aws a integrarse con AWS Organizations)
- SCP deben tener Allow Explicito (no permite nada por defecto)
- Los Deny Explícitos tienen prioridad que los Allow Explícitos
- Casos de uso:
	Restringir el acceso a ciertos servicios
	Cumplir PCI compliance (explicitamente desabilitar servicios)

### Mover Cuenta de una Organización a Otra
- Quitar la cuenta member de la Org vieja, enviar invitación desde la nueva Org, Aceptar la invitación
- Si queremos que la Master de la Org vieja se una a la nueva, primero hay que quitar las cuentas Member, y hacer el proceso anterior

## Multi Account  

- No hay necesidad de usar IAM Credentials, se hace por STS usando Iam Roles que pueden ser Asumidos en Cross Account
- CodePeline: puede invocar Codoeploy en otras cuentas
- AWS Config: Aggregators
- CloudWatch Events: Event Bus
- Cloudformation: StackSets

- Para enviar los Cloudwatch logs a una sola cuenta:

	LogGroup 	--->	 ||  	Log Destination 	---> 	Kinesis Firehouse 	---> 	S3

## Control Tower  

- No hay cargos adicionales por usar Control Tower
- AWS Control Tower runs on top of AWS Organizations
- Levantar ambientes con Compliance Multi-Account AWS Environment basado en buenas prácticas y con un par de clicks
- Automatizar el manejo de políticas usando Guardrails
- Detectar policy violations y remediarlas

- **Account Factory**:	
	- Usar Service Catalog para aprovisionar las cuentas
	- Automatiza el aprovisionamiento y despliegue de Cuentas usando **Blueprint** (CF)
	- Útil para crear baselines preaprovados y opciones de configuración para las cuentas (VPC, Subnets, Regions...)

- **Guardarils**:
	- Provee governance para controlar las cuentas de Control Tower
	- **Preventive**: Usando SCPs
	- **Detective**: Usando COnfig 

- **Landing Zone**: Combinación de AWS Organizacion, Account Factory, OUs, SCPs, IAM Identity Center, Guardrails

- **Customizations for AWS Control Tower CfCT**
	- Framework creado por AWS para customizar la Landing Zone usando CF y SCPs, usando el estilo GitOps

## AWS IAM Identity Center (Sucesor de AWS Single Sign-On)

- **Un solo login** para todas las Cuentas en una Organización, Aplicaciones de Negocio, Aplicaciones SAML 2.0, Instancias EC2 Windows...
- Se puede integrar con AD, OneLogin, Okta...
- Por medio de **Permission Set** se limitan los permisos que tendrán los usuarios/grupos de IAM Identity Center.
- Cuando se usa con Active Directory, primero hay que crear usuarios y grupos en Identity Center exactamente como están en AD.
- Usa ABAC (permisos usando Tags)
- Soporta MFA (`Always-On` y `Context-Aware`)

## AWS Tag Editor
- Permite administrar los tags de múltiples recursos a la vez

## AWS QuickSight
- Servicio para crear Dashboard de forma Serverless usando ML

## AWS Glue
- Servicio Serverless Administrado de ETL (Extractm Transform, Load)
- Útil para preparar y transformar data para analytics
- CU: Tranformar data en CSV a formato **Parquet** para luego analizarla en Athena de manera más eficiente.
- **Glue Data Crawler** Lee metadata de S3, RDS, Dynamo, JDBC y la escribe a **Glue Data Catalog**
- **Glue Job Bookmarks** Previene reprocesar data vieja
- **Glue Elastic Views** Tablas Virtuales (Materialized view)
- **Glue DataBrew** limpia y normaliza data usando transformaciones preconstruidas.
- **Glue Studio** GUI para crear, correr y monitorear Jobs ETL
- **Glue Streaming ETL** Streaming Jobs usando Apache Spark

## WAF
- Proteje las aplicaciones web en capa 7 de exploits. Se integra con LB, API Gateway, CloudFront, AppSync (GraphQL Apis)
- WAF no es para DDos, para esto se usa **AWS Shield Advanced**
- Tiene reglas Base, específicas para PHP, SQL WOrdpress, badas en reputación de IP, Bots, etc..
- Se pueden enviar los logs a CW, S3, Kinesis Data Firehose
- **Tip**: Para validar que solo se acceda a LB a travéz de CloudFront se podría agregar un Custom Header en CloudFront con un secret value y luego crear una regla en WAF para validar dicho header. Por último rotar el secret en SecretsManager y Lambda.

## AWS Firewall Manager
- Para manejar reglas en toda la organización basadas en Security Policies
- Security Policies constan de Reglas WAF, AWS Shield Advanced, Security Groups, Network Firewall VPC, Route 53

## GuardDuty
- Detección de amenazas usando algoritmos de ML. Da 30 días trial
- Verifica: CloudTrail Logs, VPC Flow Logs, DNS Logs, EKS Audit Logs...
- **Puebe proteger contra ataques CryptoCurrency**
- Se pueden manejar múltiples cuentas en GuardDuty
- Se recomienda integrar EventBridge para reaccionar a los hallazgos de GuardDuty y poder reaccionar usando Lambda. Ej: Bloquear una IP en SecurityGroup ó NACL, etc..
- **TIP**: Se puede habilitar GuardDuty usando CF template, pero si ya está habilitado dará error el Stack, para este escenario se debe crear un CustomResource usando Lambda, para habilitar GuardDuty si no se encuentra habilitado.

## Amazon Detective
- Analyza, investiga e identifica rapidamente la causa raíz de los problemas de seguridad o actividades sospechosas usando ML

## Amazon Inspector
- Automatiza **Security Assessments**
- Revisa **Instancias EC2** (Usando ssm agent), **Imagenes ECR** y **Funciones Lambda** contra una BD de vulnerabilidades CVE

## AWS Trusted Advisor
- Analyza la cuenta y da recomendaciones de 5 categorias: **Optimización de Costos, Performance, Seguridad, Tolerancia a Fallos, Limites de Servicios** 
- 7 Core Checks en el plan **Basic y Developer**:
  - S3 Bucket Permisions
  - Security Groups
  - IAM Use
  - MFA root account
  - EBS Public Snapshot
  - RDS Public Snapshots
  - Service Limits
- Full checks: **Business & Enterprise**
- Provee RECOMENDACIONES de múltiples servicios según los planes contratados. Ej:
  - EBS Snapshots, RDS Public Snapshots, S3 bucket Permissions, IAM Ussage, MFA on Root Account, SG
  - High CPU utilization, Cloudfront Optimization
  - Cloudformation Stacks, Dynamo DB Reads and Write Capacity, RDS checks, VPC Internet Gateway, EC2 Reserved/OnDemand Leases
  - LB Optimization, RDS Multi AZ, S3 Bucket Logging and Versioning, Route 53 Record Set Checks
  - Cost Optimization: Low Usage of EC2, Idle LB, Idle RDS, Saving Plangs, Lambdas with Timeouts 


## Kinesis
- Kinesis es una altenativa a Apache Kafka, ideal para BigData en "real-time", IoT, clickstreams
- La Data es automaticamente replicada a 3 AZ
- Hay 3 servicios asociados a Kinesis:
	- **Streams**:   streaming de ingesta de baja latencia a gran escala
	- **Analytycs**: Analisis en tiempo real de Streams usando SQL 
	- **Firehose**: Carga de Streams hacia S3, Redshift, ElasticSearch, Splunk...

- El Producer puede enviar 1 MB/s ó 1,000 mensajes de escritura POR SHARD, si no se obtiene `ProvisionedThroughputException`
- El Consumer puede leer 2 MB/s en lectura por SHARD en todos los consumers
- 24 Horas de retención de data por defecto, puede ser extendida a 7 días

	**Kinesis Producers**: Kinesis SDK, Kinesis Producer Library KPL, Kinesis Agent, CLoudwatch Logs, Terceros (Spark, Log4j Appenders, 					Flume, Kafka Connect, )
	**Kinesis Consumers**: Kinesis SDK, Kinesis Client Library KCL, Kinesis Connector LIbrary, Kinesis Firehose, AWS Lambda
						Terceros (Spark, Log4j Appenders, Flume, Kafka Connect)
- **TIP**:  
	- Kinesis Streams
		- Tiempo Real
		- Custom Code (producer / consumer)
	- Firehose
		- Casi Tiempo Real
		- Fully managed


## AWS Amplify
-  Se usa para construir aplicaciones Web y Mobile.
-  Para configurar el Backend ya se integra con varios servicios: S3, Cognito, AppSync, Api Gateway, Sagemaker, Lambda, Dynamo.. (Todo en un solo lugar)
-  Para el frontend se usan librerias de Amplify para diferentes plataformas: Flutter, Ionic, Next, Angular, React, IOs, Android..
-  Para el Deploy se usa Amplify Console y/ó Amazon Cloudfront
-  Se integra con CodeCommit para tener deployments por branch

## DMS Database Migration Service
- Para migrar DBs a hacia AWS / OnPremise, resilient, self healing
- Migraciones Homogeneas y Heterogeneas
- La tarea de migración se ejecuta en una EC2
- Como Source y Target se puede usar RDS, S3, DocumentDB, OpenSearch, Redshift, Kafka, Neptune, Redis... 
- Para hacer la conversión entre diferentes motores se usa el **Schema Conversion Tool SCT** (si se usa el mismo motor no es necesario). Ej:
	- SQL Server -> Mysql

## AWS Storage Gateway
- Es el puente entre la Data OnPremise  y Cloud
- Para almacenar la data Storage Gateway puede usar: **EBS, S3, Glacier**
- Los tipos de Storage Gateway son:
	- File Gateway
	- Volume Gateway
	- Tape Gateway
- Cache Refresh, sirve para que los usuarios en OnPremise vean los archivos que se crearon directamente en un bucket.

## EKS
- CU: Si la empresa ya esta usando K8s OnPremise o en otra Nube y quiere migrar a AWS
- Tipos de nodos:
	- **Managed Node Groups**: Nodos EC2 Manejados por AWS, Usa un ASG, Soporta OnDemand y Spot 
	- **Self-Managed Nodes**: Nodos creados por uno mismo, agregados a un ASG, Soporta OnDemand y Spot
	- **Fargate**: Serverless
- Container Storage Interface (CSI) soporta: **EBS, EFS, FSx for Lustre, FSx for NetApp ONTAP**
- Se puede configurar el ControlPlane para activar el Logging y enviarlo a Cloudwatch Logs. Para enviar el de los nodos o containers se debe instalar el Cloudwatch Agent y/ó usar Fluentd

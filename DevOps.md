# CodeCommit

- Para restringir que suban código a master, se debe bloquear a nivel de IAM User con política Deny
- **CodeCommit Notifications**:  Permite definir notificaciones básicas (para notificaciones y triggers avanzados usar Cloudwatch Events)
- **CodeCommit Trigger**: Permite Invocar Lambdas desde CodeCommit ( Útiles para revisar credenciales quemadas, Convenciones, etc..)

# CodeBuild (buildspec.yml)

- En CodeBuild todo el Serverless, solo se para por el tiempo que dure el build
- Al Crear el proyecto se define la imagen a usar para hacer los Builds: AWS Managed Image o Custom Image, así como los recursos y si queremos que corra adentro de una VPC. Lo minimo son 3GB y 2 vCPUs. Las imagenes de AWS son genéricas no incluyen frameworks.
- El archivo de configuración se compone de: Variables, Fases, Artefactos
- Dentro del archivo yaml se puede configurar el runtime-version ejemplo: `docker: 18`  si queremos crear y subir imagenes de docker, acá si se pueden usar runtimes de frameworks o lenguajes de programación.

# CodeDeploy (appspec.yml)

- CodeDeploy permite desplegar en EC2/Onpremise, Lambda, ECS
- Para usar CodeDeploy se debe instalar el agente de CodeDeploy en las instancias, crear un bucket para los artefactos y hacer push de los artefactos en ese bucket
- **Deployments Groups**: Son grupos de instancias (ó nodos onPremise) donde se hará el Deploy
- **Hooks**: Pasos de cada etapa del Deploy: ApplicationStop, DownloadBundle, BeforeInstall, Install,ApplicationStart, ValidateService, etc..
- Los Hooks varían según el tipo de la aplicación EC2, Lambda, y  el tipo de Despliegue: BlueGreen, In-place, etc.. 


### Deployment Configurations: 
- Donde se define el tipo de Deployment, para EC2 tenemos los siguientes:
	- In-Place:
		- AllAtOnce
		- OneAtATime
		- HalfAtATime
		- Custom (Ejemplo: Mínimo que el 80% siempre esté disponible al ir desplegando)
	- Blue/Green:
		- Se usan en conjunto con Auto Scaling Group ó con Instancias Fijas pero se deben crear antes de hacer el deployment
		- En BLueGreen el LB es necesario

- Para lambda existen los siguientes:
	- AllAtOnce, 
	- Canary: **TWO INCREMENTS** Ej. Mandar el 10% del tráfico por 10 minutos, y si todo va bien, el 100% del tráfico pasa a la nueva versión
	- Linear: **EQUALS INCREMENTS** Ej. Mandar incrementos de 10% del tráfico cada 10 minutos (se completaría en 100 minutos ó 10 veces) 

### Rollbacks
- Los rollbacks se pueden hacer manuales ó automáticos, los automáticos pueden ser: 
 1. basados en que si Falla el Deploy
 2. Basados en Umbrales de alguna alarma de Cloudwatch (Ej, si el cpu de una nueva instalación sobrepasa el 75% hacer rollback)

### On premise Instances:
- Se pueden configurar instancias fuera de AWS para desplegar aplicaciones usando CodeDeploy, en la documentación se muestran 11 pasos para realizarlo, pero básicamente hay 2 formas: Una es creando un IAM User por cada instancia y configurar cada instancia con aws cli y CodeBuild Agent esta forma es ideal cuando hay pocas máquinas OnPremises, y la otra que es más segura y trabajosa es usando un Rol haciendo uso de credenciales temporales usando STS.


# Code Pipeline:

- Cada etapa del pipeline puede crear Artifacts, los almacena en S3 y es la entrada para la siguiente etapa
- Por cada branch se debe crear un pipeline
- Para iniciar el pipeline se puede de 2 formas: 

Automático con CloudWatch Events` ó `Periódico con CodePipeline (Poll)`
- Los artifacts de CodePipeline son diferentes de los artifacts de CodeBuild
- Se pueden agregar ManualAprovals dentro de un paso del Pipeline para intervención humana (Deploy to Prod en un solo pipeline)
- Cuando un pipeline se define por CF (código), el atributo `runOrder` en los StageAction indica el órden de ejecución
- Se puede ejecutar un lambda desde el pipeline y enviar parámetros, Ej. Hacer un Request 200 al sitio web después del deploy
- `PutJobSuccessResult`, `PutJobFailureResult` para indicar al Pipeline si el Job se ejecutó con éxito o falló
- Un buen CU para CodePipeline es desplegar templates de CloudFormation, también con CF se pueden crear pipelines 

# Code Star:

- Es como Code Pipeline pero con un UI más sencillo y con plantillas definidas por lenguaje de programación y/ó frameworks
- Tiene un dashboard donde se puede ver los builds, monitoreo, jira, etc..
- Por debajo usa CF, CodeCommit, CodeBuild, CodePipeline, según la plantilla que se usó.
- En las opciones del proyecto de CodeStar se puede configurar y ver todo en el formulario: buckets, roles de servicio, CF.. 
- A bajo nivel CodeStar está configurado con un CF template.yml usando una transformación `AWS::CodeStar`


# Jenkins:

- Jenkis puede reemplazar CodeBuild / CodeDeploy / CodePipeline y/ó trabajar con alguno de estos, pero esta solución no es serverless
- **EC2 Plugin**: Lanza EC2 para levantar agentes dinámicos.
- **CodeBuild Plugin**: Usa CodeBuild en lugar de agentes fijos, volviendo un poco serverless la solución.
- **ECS Plugin**: Usa ECS en lugar de agentes fijos, volviendo un poco serverless la solución.

# Code Artifact

- Para almacenar dependencias: Maven, Gradle, npm, yarn, pip...
- Tambien funciona como proxy para los repositorios centrales y almacena una copia.
- Se le puede dar acceso a otras cuentas por medió de IAM Policies.
- **Upsteam Repositories**: Descargar dependencias de un solo endpoint y el Upsteam apuntar hasta a 10 repositorios.
- Los Domains sirven para agrupar repositorios, lo ideal es que en una sola cuenta este CodeArtifact y por medio de dominios demos acceso a otras cuentas de AWS.
- En un lambda se puede agregar usando anotaciones (decorator) ó tambien desde la consola de aws.

# Code Guru

- Automatiza las revisiones de código, usa ML
- Da 2 funcionalidades: **CodeGuru Reviewer** (analisis estático) y **CodeGuru Profiler** (performance en ejecución)
- Soporta Java y Python
- Se integra con Github, Bitbucket, CodeCommit
- Puede ser usado en aplicaciones corriendo en AWS o OnPremise
- **CodeGuru Reviewer Secrets Detector** Usa ML para detectar secrets, passwords, llaves, etc. en el código

# EC2 Image Builder

- Automatizar la creación, mantenimientoy validación de AMIs o Container Images
- Puede publicar AMIs a multiples regiones y cuentas
- Con **AWS RAM** (Resource Access Manager) se pueden compartir Images, Recipes, Components a otras cuentas
- Después de construir una imagen se recomienda almacenar el AMI ID en SSM Parameter Store, para que los cloudformation tengan siempre la última versión de las AMIs

# AWS Amplify

-  Se usa para construir aplicaciones Web y Mobile.
-  Para configurar el Backend ya se integra con varios servicios: S3, Cognito, AppSync, Api Gateway, Sagemaker, Lambda, Dynamo.. (Todo en un solo lugar)
-  Para el frontend se usan librerias de Amplify para diferentes plataformas: Flutter, Ionic, Next, Angular, React, IOs, Android..
-  Para el Deploy se usa Amplify Console y/ó Amazon Cloudfront
-  Se integra con CodeCommit para tener deployments por branch


# Cloudformation:

- En CF hay una opción que lleva al Calculator para estimar el costo mensual basado en un template
- `!FindInMap [MyRegionMap, !Ref "AWS::Region", 32]`
- Para hacer referencias entre stacks se usan los `Outputs e !ImportValue` respectivamente
- El log de un script UserData se almacena en `/var/log/cloud-init-output.log`
- `Fn:Base64` sirve para codificar un string en base64 ideal para el UserData de una EC2

- **cfn-init**: 
	- Correr scripts de una manera más elegante, EC2 se comunica con CF para bajar los scripts en formato yaml
	- Los script se crean mediante: `AWS::Cloudformation::Init` y permiten definir (packages, files, commands, sevices, etc.. como Ansible
	- Para llamar cf-init se hace desde el UserData:
		`/opt/aws/bien/cfn-init -s ${AWS::StackId} -r MyInstance --region ${AWS::Region}`
	- Los logs de cfn-init se almacenan en `/var/log/cfn-init.log`
	- Para que el template se quede esperando el resultado de cfn-init se usa `cfn-signal`
		`/opt/aws/bien/cfn-signal  -e $? SampleWaitCondition ...`

- En la CREACIÓN del stack por defecto `Onfailure=ROLLBACK`  tambien se puede usar `OnFailure=DO_NOTHING`, para la actualización siempre se hace rollback en caso de fallos
- Nunca actualizar un Nested Stack directamente, siempre hay que actualizar el Stack Padre.

- **ChangeSet**: Sirve para ver los cambios que se harán a un Stack, luego se pueden aplicar. Para estar más seguro.

- **Deletion Policy**
	- `DeletionPolicy=Retain` se puede aplicar a cualquier recurso para no borrarlo si se borra el stack
	- `DeletionPolicy=Snapshot` se puede aplicar a ciertos recursos para sacar Snapshots antes de borrarlos (Backup de la Data)
	- `DeletionPolicy=Delete` Default para todos los recursos (Excepto RDS::DBCluster que es Snapshot)
	- Para borrar un Stack que tiene un bucket primero se tiene que borrar el contenido del bucket

- **Termination Protection**: Para prevenir que eliminen el Stack. (Opción manual como en Ec2)

- **Parameters from SSM**
	- Los parámetros del Stack pueden ser de SSM: `Type: AWS::SSM::Parameter::Value<String>`
	- Si se actualiza el stack usando la misma plantilla y los parámetros de SSM cambiaron, se actualizan los recursos afectados por el cambio de parámetros.
	- Hay parámetros públicos de AWS que se pueden usar en los template:
		`'/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'`

- **DependsOn**: Este recurso debe crearse hasta que X recurso se cree primero

- Para desplegar Lambdas en cloudformation, se puede subir el código a S3 y además especificar la versión del Zip de S3 (`S3ObjectVersion`) esto con el fin de que al cambiar el tag versión CF desplegará los cambios ya que el template cambiará

- **Custom Resources**:
	Se pueden crear recursos custom en cloudformation `Custom::MyCustomResource` que hacen referencias a funciones Lambda. Estas funciones son invocadas cuando ocurren eventos en el Stack de CF como por ejemplo: CREATION, DELETE_IN_PROGRESS, ETC. 
	En el ejemplo se creó un recurso Custom que hacía referencia a un lambda que limpiaba un bucket cuando el Stack entraba en DELETE_IN_PROGRESS, así CF no daría error de que el Bucket debía estar vacío para borrar el Stack.

- **Drift Detection**:
	Ayuda a detectar si los recursos creados con el Template aún están igual (IN_SYNC) o no (DRIFTED), muestra las diferencias si no lo estan en una tabla y en formato JSON

- **UPDATE_ROLLBACK_FAILED**: Leer documentación adicional para este estado

- **InsufficientCapabilitiesException**: Cuando no se han puesto los capabilities necesarios. Leer documentacion adicional

- **cfn-hup**: 
	- Levanta un Daemon en EC2 que se queda escuchando si hay nuevos cambios en la Metadata de EC2
		`/opt/aws/bincfn-hup ...`
	- `/etc/cfn/cfn-hup.conf` almacena el nombre del stack y las credenciales para el Daemon 
	- El Intervalo de chequeo para cambios en la metadata es de 15 minutos, pero se puede cambiar
	- Ejemplo: Verificar cada 2 minutos si la metadata `CloudFormation::Init` ha tenido cambios, y si ha tenido ejecutar un script

- **cfn-get-metadata**: Script para obtener la metadata

- **Stack Policies**:
	- Son como políticas de IAM donde podemos restringir acciones sobre recursos específicos del Stack
	- Se especifican en las opciones avanzadas cuando se está creando el Stack
	- Estas políticas se validan cuando se actualiza el Stack
	- Aunque se puede modificar la política temporalmente para permitir acciones específicas, posiblemente por modificaciones urgentes, pero luego de aplicar la modificación, la política regresa a su estado original con el que fué creada.


# Elastic Beanstalk

- En el exámen pueden preguntar si la BD debe crearse como parte del ambiente de Beanstalk ó por aparte, depende si queremos que la BD forme parte del mismo ciclo de vida de la aplicación de Beanstalk, ya que si queremos conservar la BD aunque se elimine el ambiente lo mejor es crear la BD por separado.

- **Deployments**
	1. All at once:
		Es la forma más rápida, pero hay downtime 
	2. Rolling:
		Actualizar pocas instancias al mismo tiempo (bucket) y luego actualizar el siguiente bucket una vez el primer bucket está healthy. Acá no hay downtime pero se reduce la capacidad mientras se despliega, el tamaño del bucket se puede setear.
		También durante el despliegue las 2 versiones coexisten.
	3. Rolling With Additional Batches:
		Como la anterior solo que se crean nuevas instancias para desplegar el batch, de este modo las instancias viejas siguen disponibles durante el despliegue permitiendo operar a la capacidad máxima. También se puede setear el tamaño del bucket
		También durante el despliegue las 2 versiones coexisten. Al final de actualizar las instancias las que se crearon se borran
		Bueno para producción
		Como levanta instancias para procesar el batch es un poquito más caro que Rolling
	4. Inmutable:
		Levantar nuevas instancias en un nuevo ASG, desplegar la versión en esas nuevas instancias, y luego cuando esas instancias están healthy las mete al ASG original y finaliza la versión vieja
		Aca no hay downtime
		Es la opción más cara
		En caso de fallas es la más rápida de hacer rollback, solo se finaliza el nuevo ASG
		Bueno para producción
	5. Blue/Green
		No es un feature de Beanstalk, es más trabajo manual
		Lo que busca es que no exista Downtime
		Crea un nuevo ambiente "Stage" y despliega la nueva versión ahí, la nueva versión (green) puede validarse y hacer rollback si no cumple, Luego con Route 53 (Weighted Policies) se le manda un porcentaje pequeño de carga Ej 10%, y al final usando Beanstalk usar la opción "Swap URL" para intercambiar URLs entre ambientes

- **Workers Environments**
	- Ambientes en EB que se utilizan para correr tareas de larga duración o calendarizadas. (Encode a video)
	- En EB se puedeb crear ambientes de 2 tipos: Web Server, Worker
	- Este ambiente crea 2 colas en SQS para procesar trabajos: `WorkerQueue` y `WorkerDeadLetterQueue`, adicionalmente crea un archivo cron.yaml, para ejecutar tareas calendarizadas.
	

# LAMBDA

- Trust Relationships (En Roles de IAM): Son los servicios de AWS que pueden asumir un rol.
- El CPU es asignado acorde a la cantidad de memoria
- El tiempo máximo de ejecución es 15 minutos. Si una tarea se tarda más de eso lo mejor sería usar AWS batch
- Se puede configurar un DLQ para cuando Lambda falla más de 3 veces, solo aplica para funciones asincronas
- Por defecto son 1000 lambas por 'Reserve Concurrency', se puede incrementar.
- Por temas de compliance se puede activar el log de las invocaciones en CloudTrail
- Aperte de Api Gateway tambien se puede integrar directo con ALB (pero se pierden los features de de api gateway, ej. autenticacón)
- Lambda por si mismo integra KMS para tener variables encriptadas en tránsito, y en la función se debe hacer el decrypt en ejecución, y en 	código ya no es necesario indicar la llave criptográfica porque ya se configuró a nivel de lambda
	`boto3.client('kms').decrypt((CipheredtextBlob=b64decode(os.environment["DB_USER"])))`
- Una versión en un lambda engloba el código y la configuración, nada se puede cambiar (Versión = INMUTABLE)
- Alias es un puntero a una versión (Alias = MUTABLE)
- A nivel de Alias se puede usar una segunda versión adicional con un % de tráfico asignado.
	CANARY = TWO STEPS (ej: primero 10%, luego 100%)

	### SAM

	- Para correr la función local:
		`sam local invoke "HelloWordFunction" - e events/event.json` 
	- Para levantar el API Local:
		`sam local start-api` 
	- Para generar el cloudformation template
		`sam package ...`

	- SAM se integra con CodeDeploy para poder hacer Canary Deployments, solo es necesario agregar unas líneas a la configuración de la función Lambda. Al hacer esta integración se crea una aplicación en CodeDeploy
	Utiliza Alias para esta integración, permite CANARY deployments y permite hacer rollback y configurar Alarmas, también permite llamar funciones Hooks para validación del deployment. LEER LINK.
	
- Step Function también se puede usar en conjunto con AWS BATCH para orquestar los Jobs cuando son tareas muy largas.
- Otro caso de uso para Step Functions es para Flujos automatizados inclusive con intervención humana.
- Los flujos de StepFunctions pueden durar hasta 1 año
- El Tip para el examen es que StepFunction sirve para orquestrar cosas como flujos
- A nivel de Cloudwatch Rules se pueden configurar eventos basados en las transiciones y estados de StepFunction
- En la consola se puede ir viendo las tranciciones en el flujo, el input y output de cada paso, etc..


# API GATEWAY

- El tipo de EndPoint puede ser Regional (Toda la región), Edge Optimized (Menor latencia usando todas las Edge Locations), 
	Private (Desplegada dentro de la VPC y para acceder a ella es necesario un VPC Endpoint)
- Api Gateway se puede integrar con Lambda, cualquier endpoint HTTP, Mock, AWS Service, VPC Link
- Se puede hacer rollback de un Stage ya que el historial se almacena
- Los Stage se pueden usar para manejar 2 versiones simultaneamente /v1, /v2  
- Las Stage Variables son como variables de ambiente pero para API Gateway, pueden ser usados en Lambda ARN, HTTP Endpoints, Parameter Mapping Templates, también son pasadas a la función Lambda a travéz del objeto `context`
- Las Stage Variables se usan comunmente para indicar a que Alias de las funciones lambda llamar.
- A nivel de Stage se pueden habilitar Canary, y asignarle un porcentaje del tráfico a los nuevos deployment del Stage, luego después de la aceptación se hace Promote al Canary y todo el tráfico se mueve a esa versión del deployment.
- API Gateway tien eun límite de 10k RPS a nivel de CUENTA (Se puede incrementar), `429 Toomany Requests`
- Con los 'Usage Plans' se puede limitar el número de RPS para un API Key. Se puede limitar por Segundo, Por Mes, y esta configuración se puede granularizar a nivel de API + Stage + URL + Method.
- **Tip Examen**: Es posible tener Throttles a nivel de API Gateway, a nivel de Usage Plan y a nivel de Lambda.
- Usando la integración de `AWS Service` podemos llamar desde API Gateway a un StepFunction, en el json body del Request deben ir 3 propiedades: input (json del step function), name, stateMachineArn. La respuesta es async y nos indica que el Statemachine ha sido iniciado.


# ECS

- ECS Clusters es una agrupación logical de instancias EC2
- Las instancias corren el agente de ECS y usan una AMI especial para ECS, el agente es un contenedor de docker
- 1vCPU = 1024 CPU Units in ECS
- ECS crea automáticamente un grupo de escalamiento para subir la cantidad de nodos
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
	 - Amazon lanza Fargate para que no se tengan que aprovisionar EC2, todo queda del lado de amazon y Serverless
	 - Para escalar solo se incrementa el numero de Task, Simple!
	 - No soporta HostPort como al desplegar en EC2.
	 - También soporta los Deployments: Rolling Update y Blue/Green


	 ### ECS + Beanstalk
	 - Se puede correr Elastic Beanstalk en modo Single y Multi Docker Container 
	 - Multi Docker ayuda a correr multiples containers por EC2 en EB
	 - Por debajo crea: ECS Cluster, EC2 Instances, ASG, LB, Task Defintions
	 - Requiere un Dockerrun.aws.json en los fuentes, y la imagen debe ser pre subida en ECR

	 ### Roles en ECS 
	 - **Classic**: Cuando se crea un clúster ECS Classic, a las instancias se les asigna un Rol el cuál tiene los permisos para modificar el clúster, descargar imagenes de ECR, etc. Cuando se lanzan Services el Service tiene un TaskDefinition y ese a a su vez tiene un TaskRole, ese TaskRole tiene un TrusRelationship para que solo pueda ser asumido por ECS y es el que va a tener los permisos que necesitan los contenedores /aplicación. 
	 - **Fargate**: Cuando se crea un clúster ECS Fargate, como no hay instancias solo se crea el Rol TaskRole. (Más simple)

	 ### Autoscaling
	 - En ECS se configura a nivel del Servicio, minimo, máximo, Scaling Policy, etc..
	 - Hay 2 políticas de autoescalamiento:
	 	- **Target Tracking**: Se monitorea en base a una métrica Ej: CPU y en base a eso se incrementan los Task
	 	- **Step Scaling**: Basada en una Alarma (CPU, Memoria, etc..)  y en base a eso se incrementan los Task
	 	La diferencia en una y otra es que la segunda usa Alarmas.

	 - Hay un problema con el autoescalamiento si el clúster es Classic, ya que lo que se incrementa son los Task, y no las instancias. 
	 - El escalamiento de ElasticBeansTalk con ECS, si incrementa las instancias y los Task, en fargate no se tiene este problema ya que teoricamente no hay instancias.

	 ### Cloudwatch Integration
	 - A nivel de Contenedor se puede definir la configuración de Log, se le puede indicar el driver (awslogs, splunk), si es awslogs se pueden enviar directo a Cloudwatch Logs.
	 - Si se usan instancias de EC2 entonces se debe instalar el Agente de Clouddatch y enviar los archivos del SO a Cloudwatch Logs por ejemplo `/var/log/docker` 
	 - A nivel del cluster de ECS tambien se puede activar el `Cloudwatch Container Insights` para colectar Métricas (CPU, memoria, Disk, etc.. A NIVEL DE CONTENEDOR y enviarlos a Cloudwatch, para utilizarlo en las métricas, dashboard, etc.. (Tiene un costo adicional)
	 - Con Cloudwatch Events se pueden configurar reglas basadas en los eventos de ECS ej: cuando se termina un container, quiere levantar otra tarea, enviar una alarma, ejecutar un comando, etc..


# OpsWorks

- OpsWorks consta de 3 partes: OpsWorks Stack, OpsWorks for Chef Automate, OpsWorks for Puppet Enterprise
- Es para las personas que usan Chef OnPremise y luego quieren migrar a la nube y seguir usando Chef
- Un Stack es un grupo de Layers, Instaces and AWS Resources.
- Los tipos de Layers que se pueden crear son: 
	- Opsworks
	- ECS
	- RDS
- Para iniciar se utiliza Chef cookbooks
- Tipos de instancias:
	24x7: Instancia Normal
	Time: Permite definir horarios para levantar y bajar las instancias en base semanal.
	Load: Estas se levantan o bajan automaticamente en base a la carga (CPU, memoria, etc..) Como Autoescale pero no tan flexible
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

- **Auto Healing**: Si el agente de OpsWork no se puede comunicar con AWS en 5 minutos, la instancia es reiniciada. Con CloudwatchEvents podemos definir una regla para ser notificados cuando esto pase.


# CloudTrail

- El trail puede capturar los eventos de todas la regiones
- CloudTrail envía los archivos a s3 pero también se puede configurar para enviarlos a CloudWatch Logs (Ambos al mismo tiempo)
- Al integrarlo a CloudWatch Logs se pueden crear métricas
- Por defecto los archivos se encriptan usando SSE-S3
- Estructura del Log:
	- `userIdentity`: Quién hizo el request
	- `eventTime`: Cuándo
	- `eventSource`: Dónde
	- `eventName`: Qué
	- `requestParameters`: Parámetros adicionales
	- `responseElements`: Respuesta
- Puede haber un Delay de hasta 15 minutos por parte de CloudTrail, para eventos en tiempo Real se recomienda usar Cloudwatch Events
- El digest de los archivos aparece cada hora
- **Integridad de los logs**: Con el siguiente comando podemos ver si un archivo fue alterado ó eliminado
	`aws cloudtrail validate-logs --trail-arn ... --start-time ... `
- Para mandar los archivos de Cloudtrail a un S3 de otra cuenta (CrossAccount), hay que modificar el BucketPolicy para permitir escrituras y lecturas de otras cuentas.


# Kinesis

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


# Cloudwatch

- La información de las métricas depente del intervalo de tiempo que tengan por ej: 
	Métricas que se reportan cada minuto la data se almacena por 3 horas, y la que se reporta cada hora está disponible 15 meses.
- RAM, Espacio en Disco, y Número de procesos, son métricas que solo se obtienen al usar Custom Metrics ó el Agente de Cloudwatch
- A nivel ASG se pueden activar "Group Metrics" para ver métricas de EC2 agrupadas por ASG
- En las categorías de las métricas ej: "EC2" se puede dar a la opción "Automatic Dashbaord" crea un dashboard las métricas importantes
- **Custom Metrics**
	Standard Resolution: 1 minuto de granularidad.  (Las métricas de AWS son Standard Resolution por Default)
	High Resolution: : 1 segundo de granularidad
	`aws cloudwatch put-metric-data --metric-name FunnyMetric --namespace Custom --value 123 --dimensions InstanceId=1123,...`
- **Exportar Métricas**:
	`aws cloudwatch get-metric-statistics --dimensions --start-time --end-time ...`
	Desd el punto de vista de devops lo ideal es automatizar el comando anterior, invocar un lambda periódico desde Cloudwatch Rules y en el lambda con el SDK ejecutar el get-metric-statistic y escribir a S3 ó ELASTICSEARCH
- **TIP**: How to Correlate Data? "Cloudwatch Dashboard"

## Cloudwatch Alarms
  - Las acciones de una alarma pueden ser Notificaciones (SNS), AutoScaling, EC2 ACtions, System Manager Action
  - Una Cloudwatch Alarm no puede ser un evento de entrada para un CLoudwatch Events
  - En us-east-1 se puede crear una Billing Alarm, basado en el total estimado ó por servicio

## Cloudwatch Logs
  - Para configurar la Retención de los Logs se hace a nivel de LogGroup
  - Se pueden obtener metricas y logs de Instancias Onpremise con el CloudWatch Agent
  - El agente tiene un wizard y forma un json que puede ser almacenado en SSM ParameterStore para que las instancias lo bajen de ahí.
  - Métricas que se obtienen con el agente:
	cpu, disk, space, io, ram, network, processes, swap
  - **Metric Filter**, sirve para buscar coincidencias en Cloudwatch Logs.
  - De un MetricFilter se puede crear un Metric y del Metric un MetricAlarm
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


# EventBridge

- Previamente conocido como CloudWatch Events
- Adicionalmente tiene eventos de Otros Partners (Symantec, Sugar CRM, Datadog...)
- **CloudTrail Integration**: 
	Para trabajar con eventos más específicos, se debe seleccionar el Servicio Ej: EC2 y en el tipo de evento se debe 	seleccionar `AWS API Call via CloudTrail` y acá ya podemos usar eventos más específicos como por ej: `CreateImage` (Las operaciones List, Get, Describe no son soportadas por EventBridge). En el ejemplo se creaba una alarma cuando crean un AMI
- **S3 Integration**:
	Forma 1:
		Dentro de S3 en Opciones Avanzadas está la opción de Events, podemos elegir que eventos serán los disparadores (PUT, COPY, Delete, ect..) y luego "Send To" que permite llamar a SNS, SQS, Lambda. En esta forma no todos los eventos de S3 están soportados.
	Forma 2:
		Hacerlo en EventBridge, pero primero hay que habilitar que CloudTrail grabe los eventos de Data de S3 a nivel de todos los Bucket o solo de alguno es específico. Esto para tener el soporte de Eventos a nivel de Objects de S3


# X-Ray

- **Keywords** para el examen: Traces, Debbuging, Distributed application
- Se puede automatizar con Lambda la detección de latencia, errores, etc, en una aplicación (Ver link del DevOps Blog)


# SSM

- Ayuda a administrar EC2 y sistemas On-Premise a escala, detectar problemas, parcheo. Funciona para Windows y Linux
- Servicio gratuito integrado con CloudWatch, AWS Config
- El agente viene por defecto instalado en las Linux AMI y Ubuntu AMI
- **TIP**: si una instancia no se puede controlar desde SSM, puede ser un problema con el agente ó con el Rol 

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


# CONFIG

- Util para dar seguimiento a la configuración de todos los recursos de una cuenta
- Cada regla que se agrega tiene un costo de $1 al mes
- Las reglas se pueden validar siempre que hayan cambios o de forma periódica
- También tiene Remediation Actions
- **CustomRules**: 
	- Se configuran usando Lambda
	- Pueden ser invocadas por cambios en la configuración o de forma periódica
	- Puede monitorear cambios en los tipos de recursos asociados, Tags, Todos los cambios
- Por cada regla se puede definir un Remediation Action (Por debajo usa SSM Automation usando las de AWS o definiendo un Custom RemediationAction). Por cada RemediationAction se definen Retries, timeout, parámetros de entrada,

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

**TIP** Leer a fondo Cloudwatch Events


# SERVICE CATALOG

- Service Catalog es simple y restringe muchas opciones a los usuarios
- Tareas del Admin: Crear Productos (CF Templates) -> Crear Portfolio (Colección de productos) -> Control IAM Permissions to Porfolios
- Tareas del Usuario: De un Product List -> Usar Productos Aprovisionados que ya se encuentran correctamente configurados.
- Ayuda a Governance, Compliance y Consistency
- Ayuda a los usuarios a lanzar productos (CF Stacks) sin tener mucho conocimiento
- Los Output de CF son super importantes porque es la forma en que los usuarios verán los endpoints creados, direcciones, etc..

# INSPECTOR

- Inspector busca vulnerabilidades por Instancia, Container y Accesibilidad de Red
- Para hacer un escaneo a una instancia, es necesario que SSM esté activado y las instancias tengan el Rol de 
	`AmazonSsmRoleForInstancesQuickSetup`


# EC2 COMPLIANCE

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



# PERSONAL HEALTH DASHBOARD

- Se integra con Cloudwatch Events para notificar o realizar acciones en base a los eventos de Health de todos o algunos servicios que afectar a la cuenta.
- Cuando un AccessKey y SecreteKey son expuestas en un repositorio de Github público, AWS monitorea y envia un evento que puede ser Capturado en CloudwatchEvent y realizar una tarea automatizada. En el ejemplo se llama un StepFunction que elimina las credenciales, hace un query en Cloudtrail para ver que hicieron con esas credenciales, luego notifica por correo electrónico el resultado de cloudtrail.


# TRUSTED ADVISOR

- Da recomendaciones para la cuenta, tiene 2 tier, la segunda tiene más recomendaciones y es más cara. Se centra en:
	- Cost Optimization: Low Utilization EC2 Instances, Idle LB, Underutilized EBS VOlumnes, Unassociated IP Addreses, ...
	- Performance: Alta utilización de Instancias, EBS Volumnes..
	- Security: MFA en root, Security Groups, Public Snapshots, S3 Permissions, IAM Access Key Rotation 
	- Faul Tolerance: Chequea la edad de los Snapshots, Optimización de LB, Redundancia de Tunel de VPN, RDS Backups...
	- Service Limits: Chequea cuando el se llega al 80% de los límites de varios servicios
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



# GUARD DUTY

- Detecta amenazas de forma inteligente en la Cuenta usando Machine Learning
- Analiza CLoudTrail Logs, VPC Flow Logs, DNS Logs
- Notifica en cazo de algún hallazgo y se integra con AWS Lambda
- Puede detectar si una instancia está minando Bitcoin
- Detecta Ataques de fuerza bruta
- Tambien se integra con Cloudwatch Events para automatización, el evento es `GuardDuty Finding`. Es importante hacer estás integraciónes para ser notificados ó tomar accones en tiempo real.
- 

# MACIE

- Ayuda a analizar data sensible en S3 y da insights acerda de ella.
- Analiza si en la data hay:
	Private Keys (DSA, EC, PGP, RSA) 
	Encrypted Data Blocks
	Password etc shadow
	Secret Keys (Facebook, Github, Slack, etc..)
- Tiene integración con cuentas externas
- Se puede customizar varias cosas (extensiones de archivos, expresiones regulares, etc..)

# SECRETS MANAGER

- La principal diferencia con Parameter Store, es que acá se pueden rotar los secrets y se integra con RDS.
- $0.40 por secret por mes


# LICENSE MANAGER

- Para manejar el licenciamiento de varios proveedores: Microsoft, Oracle, Sap, etc..
- Se pueden definir reglas para aumentar el número de licencias.
- Se pueden asociar AMIs a la reglas, así cuando se lancen instancias de esa AMI, se obtiene una licencia del Pool definido.

# COST ALLOCATION TAGS

- Son como los Tags, pero se muestran como columnas en los Reportes
- Tipos de tags: 
	**User Tags**:
		Definidos por los usuarios (los tags normales)
		Inician con el Prefijo user:
	**AWS Allocation Tags**:
		Solo se muestran en el Billing Console
		Tiene el prefijo aws: Ej: `aws:createdBy`
		NO son aplicadas a los recursos que se crearon antes de la activación
	**Cost Allocation Tags**:
		Solo se muestran en el Billing Console
		TOman hasta 24 horas para aparecer en los reportes
		NO son aplicadas a los recursos que se crearon antes de la activación
		Para activar estos Tags, de los Tags Normales se seleccionan cuales van a ser tambié nCtostAllocationTag


# TIPS

- Todos los endpoints de AWS exponen HTTPS
- S3 si tiene un endpoint HTTP, pero no se debería usar

S3 Data Encryption At Rest
- SSE-S3: Usando llave de AWS
- SSE-KMS: Usando propia llave de KMS
- SSE-C: Usando propia llave (AWS no la almacena)
- Client Side encryption: Enviar data encryptada a AWS, (AWS no tiene conocimiento de la llave)
- Glacier usa encriptación por defecto

Encripción de EBS, EFC, RDS, ElasticCache, DynamoDB
- Se puede usar la llave del servicio o una propia de KMS

-PHI = Protected Health Information 
-PII = Personally-Identifying Information

- Direct Connect = Conexión privada directa entre site y AWS
- VPN = Site-to-Site para protejer a nivel de internet
- Network ACL: stateless firewall a nivel de VPC
- WAF: Seguridad web contra Exploits
- Security Group: Staful firewall a nivel del Hypervisor de instancia 
- System Firewall: Instalar firewall dentro de EC2 (Linux firewalls, Windows Firewall)



# ASG

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

 **Target Tracking Scaling**:
		- En esta opción se basa en métricas (CPU, LB Request COunt, Network Bytes In/Out), y cuando se excede del porcentaje deseado de todo el AG, se aumenta el "desired" creando así una nueva instancia. 
		- Aca existe una opción llamada "disable scale-in" para que las instancias nuevas que se levantan no se borren incluso si la métrica ya bajó, de este modo el AG solo va a crear instancias no a bajarlas.
		- Cuando se crea un Scalin Policy también se crea un Alarm en Cloudwatch que es el que realmente controla el AG.

 **Simple Scaling Policy**:
		- Acá primero hay que crea una alarma en Cloudwatch, luego seleccionar si queremos Subir/Bajar N Instancias.
		- Suena mas sencilla que la anterior, pero la ventaja que da es que como se baja en una Alarma propia, podría ser en respuesta de cualquier servicio de AWS.

 **Scaling Policy With Steps**:
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
 - Si se suspende el `AddToLoadBalancer` lo que hace es que agrega la instancia al ASG, pero no la agrega al TargetGroup del LB, para que no reciba tráfico. Después se puede asociar manualmente al TargetGroup para que reciba tráfico.
	(Hay que aprenderse todos estos eventos usando el Bookmark.)

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
- Default
	La default evalúa lo siguiente:
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



# DEPLOYMENTS STRATEGIES (REVIEW)

- In Place 		(1 LB, 1 TG, 1 ASG)	(dowtime, las mismas instancias se actualizan)
- Rolling  		(1 LB, 1 TG, 1 ASG, nuevas instancias)
- Replace  		(1 LB, 1 TG, 2 ASG, nuevas instancias)
- Blue / Green  (2 LB, 2 TG, 2 ASG, nuevas instancias, R53) (Acá manda el Route 53: Simple / Weighted (Mandar un poco de tráfico primero))


# DYNAMO

- Indices Secundarios Locales solo se pueden crear cuando se crea una tabla, después solo se pueden crearn secundarios globales. Además los Locales tiene que tener la misma Llave primaria que la tabla, solo el Sort cambia.
- La capacidad On-Demand (siempre va a funcionar bien) es mucho más cara que Provisioned. Cuando se selecciona Provisioned se definen los Read/Write Capacity Units ó el mínimo y máximo para escalamiento.

### DAX: 
Clúster de Dynamo a nivel de tabla para lecturas intensivas, elegimos el tipo de instancia de los nodos y cuantos queremos, para que la data más accesada se mantenga en memoria.

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


# Multi AZ in AWS

- Servicios donde Multi-AZ debe ser habilitada manualmente:
	
	EFS, ELB, ASG, Beanstalk: Asignar AZ
	RDS, ElasticCache: Multi-AZ
	Aurora: 
		Data es almacenada atravéz de multi-AZ 
		Puede tener multi-AZ para la BD (tal como RDS)
	ElasticSearch:	Multi master
	Jenkins: Multi Master

- Servicios donde MUlti-AZ es implícito:
	
	S3: (Excepto OneZone-InfrequentAccess)
	DynamoDB
	Todos los Managed Services propietarios de AWS

- EBS
	Está atado a una sola AZ. Que se podría hacer para que EBS fuera Multi AZ?

	1. Tener un ASG con 1 min/max/desired
	2. Tener un Lyfecycle hooks para Terminate: Que haga un snaphot del EBS volume
	3. Tener un Lyfecycle hooks para Start:	Que copie el snapshot, cree un EBS y lo atache a la instancia nueva

	TIP: Sí un EBS está usando PIOPS, para obtener el máximo performance después del snapshot, se debe leer el volumen entero una vez (pre-warming of IO blocks)


# Multi Region Services

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

# Cloudformations StackSets

- Para desplegar Stacks en múltiples cuentas / regiones
- Una vez desplegado el StackSet en múltiples regiones ó cuentas se pueden agregar más regiones después.

- CUs
	- Habilitar AWS Config en todas las regiones ó cuentas donde esté trabajando.
	- Habilitar CloudTrail en otra cuenta 
	- Habilitar GuardDuty en otra cuenta


# Multi Region CodePipeline
- El pipeline debe copiar los artefactos a las diferentes regiones para que así CodeDeploy en diferente región pueda encontrar los artefactos.

# DISASTER RECOVERY 

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

# On-Premise Strategies with AWS

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

# AWS ORGANIZATIONS 

- Es un servicio Global
- La cuenta principal es 'master' y las demas 'members', las members solo pueden pertener a una organización.
- El Billing se consolida a todas las cuentas.
- Se puede habilitar CloudTrail en todas las cuentas para enviar los logs a un S3 Central.
- Tambien enviar los Cloudwatch Logs a una cuenta central de Logging
- Las OU agrupan varias cuentas, pueden haber OU adentro de un OU

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


# Multi Account  

- No hay necesidad de usar IAM Credentials, se hace por STS usando Iam Roles que pueden ser Asumidos en Cross Account
- CodePeline: puede invocar Codoeploy en otras cuentas
- AWS Config: Aggregators
- CloudWatch Events: Event Bus
- Cloudformation: StackSets

- Para enviar los Cloudwatch logs a una sola cuenta:

	LogGroup 	--->	 ||  	Log Destination 	---> 	Kinesis Firehouse 	---> 	S3


# Control Tower  

- No hay cargos adicionales por usar Control Tower
- AWS Control Tower runs on top of AWS Organizations
- Levantar ambientes con Compliance Multi-Account AWS Environment basado en buenas prácticas y con un par de clicks
- Automatizar el manejo de políticas usando Guardrails
- Detectar policy violations y remediarlas

- **Account Factory**:	
	- Usar Service Catalog para aprovisionar las cuentas
	- Automatiza el aprovisionamiento y despliegue de Cuentas
	- Útil para crear baselines preaprovados y opciones de configuración para las cuentas (VPC, Subnets, Regions...)

- **Guardarils**:
	- Provee governance para controlar las cuentas de Control Tower
	- Preventive: Usando SCPs
	- Detective: Usando COnfig 

* Ver un video más completo de Control Tower




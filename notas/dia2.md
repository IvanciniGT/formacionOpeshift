
# Contenedor

Contenedor es un entorno aislado donde ejecutar procesos dentro de un SO que corre un kernel linux.
Aislado:
- Sus propias variables de entorno (env)
- Su propio sistema de ficheros (fs)
- Su propia configuración de red ( su propia IP, etc)
- Limitaciones de acceso a los recursos hardware del host (CPU, RAM, etc)

El FileSystem del contenedor, suele seguir el estándar POSIX (que es una especificación)... y se hereda de una imagen de contenedor.

Ese sistema de archivos se monta en capas:
- La capa base se hereda de la imagen (esta se comparte entre todos los contenedores que se creen a partir de esa imagen)
- Encima se montan una capa específica del contenedor, donde se guardan los cambios que se hagan en el contenedor (ficheros creados, modificados, etc)
- Podemos montar (puntos de montaje) otros volúmenes externos (nfs, discos locales, etc) en el fs del contenedor (mount)... solo que los gestores de contenedores me lo ponen fácil.

Las IPs de un contenedor se sacan de una red virtual privada (similar a la de loopback) que los gestores de contenedores crean en el host. Ahí es donde los procesos que corren dentro del contenedor abren puertos.

Desde fuera, no podemos a priori acceder a esos puertos... para resolver esto, se hace un nat a nivel del host, que mapea puertos en las IPs del host a puertos del contenedor => (Redirección de puertos).

    ---> HTTP ----> IP HOST:PUERTO_A ---> IP CONTENEDOR:PUERTO_B

    Esa redirección que se configura a nivel del host es gestionada por el gestor de contenedores (docker, podman, etc)

# Volúmenes

Para que se usan?
- Persistencia tras el borrado del contenedor
- Compartir datos entre contenedores  ~ Escalabilidad (pero también tiene otros usos)
         Contenedor Wordpress (Apache)-> access log <- Contenedor(Filebeat) -> ElasticSearch  
- Suministrar configuraciones propias.... por ejemplo inyectando archivos de configuración en el contenedor (para el nginx, mariaDB, etc)

# Configuraciones propias para los programas que corren dentro del contenedor

- Inyección de ficheros en el filesystem del contenedor (volúmenes)
- Variables de entorno (env)

---

# Kubernetes

Docker y otros gestores de contenedores nos permiten crear y gestionar contenedores en un solo host.

Pero con un host no monto un entorno de producción.

Kubernetes nos ayuda a montar entornos dee producción (donde los aplicativos corren en contenedores)

A nosotros eso nos da un poco igual. Pero desde el punto de vista de Kubernetes es la base de todo... es lo que lee permite automatizar la creación y operación del entorno de producción (por la estandarización de los aplicativos en contenedores)

La gran diferencia conceptual entre Kubernetes y herramientas tipo Nutanix / VMware NO es que Kubernetes trabaja con contenedores y VMware con máquinas virtuales (eso es anecdótico)... 
la gran diferencia es que Kubernetes opera y crea automáticamente entornos de producción estandarizados (aplicativos en contenedores).... mientras que VMware / Nutanix somos nosotros los que creamos y operamos esos entornos (máquinas virtuales).

Y la diferencia está en el lenguaje!

A Nutanix / VMware usamos interfaces gráficas o comandos para decirle cosas como:
- Crea una máquina virtual con estas características (CPU, RAM, disco, etc)
- Instala este sistema operativo
- Reinicia esta máquina virtual

A kubernetes le damos descripciones en ficheros YAML (manifiestos) que describen el entorno de producción que queremos (aplicativos en contenedores) y Kubernetes asume la responsabilidad de crear y operar ese entorno automáticamente.

Con Nutanix o VMware usamos lenguaje IMPERATIVO (dile a la máquina lo que tiene que hacer paso a paso)
Con Kubernetes usamos lenguaje DECLARATIVO (describe el estado final que quieres y kubernetes se encarga de llegar a ese estado)

Toda la configuración de Kubernetes se hace a través de ficheros YAML (manifiestos) que describen el estado final que queremos.
En esos archivos definimos (DECLARAMOS) Objetos de Kubernetes. Esos objetos son los bloques LEGO con los que construimos nuestro entorno de producción.
Kubernetes me ofrece unos 25-50 tipos de objetos diferentes (depende de la versión y de las extensiones que tenga instaladas)... distribuciones de kubernetes como OpenShift añaden sus propios objetos personalizados (CRD=Custom Resource Definition).
Los objetos más importantes son:
- Node
- Namespace
- Pod
- Deployment
- StatefulSet
- DaemonSet
- Job
- CronJob
- Service
- Ingress
- ConfigMap
- Secret
- PersistentVolume
- PersistentVolumeClaim
- StorageClass
- HorizontalPodAutoscaler
  
- ServiceAccount
- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding
- ResourceQuota
- LimitRange
- NetworkPolicy
- ...

---

# YAML

Es una alternativa a JSON o a XML para definir datos estructurados.
JSON y XML son bastante más limitados en funcionalidad y legibilidad que YAML.

YAML nace con una idea clara en mente: Facilitar la escritura y lectura de archivos de configuración por humanos.

Hay un huevo de herramientas que usan YAML:
- Docker compose
- Kubernetes
- Ansible
- Gitlab CI/CD | Github Actions
- La configuración de red hoy en día een una máquina Ubuntu se define en YAML (netplan)
- ...


---

# Arquitectura de un cluster de Kubernetes

+------------------------------------------------------------------------- red de la empresa (192.168.0.0/16)
|
+- 192.168.1.101 - NodoMaestro1
||                       Linux
||                       Crio
||----- 10.10.1.1 ---------- POD: CoreDNS (1 contenedor)
||----- 10.10.1.2 ---------- ApiServer
||----- 10.10.1.3 ---------- Scheduler
||----- 10.10.1.4 ---------- Etcd
||----- 10.10.1.5 ---------- Kube-Proxy
||                       Kubelet
+- 192.168.1.102 - NodoMaestro2
||                       Linux
||                       Crio
||----- 10.10.1.6 ---------- POD: ApiServer (1 contenedor)
||----- 10.10.1.7 ---------- Etcd
||----- 10.10.1.8 ---------- Controller Manager
||----- 10.10.1.9 ---------- Kube-Proxy
||                       Kubelet
+- 192.168.1.103 - NodoMaestro3
||                       Linux
||                       Crio
||----- 10.10.1.10 --------- Scheduler
||----- 10.10.1.11 --------- CoreDNS
||----- 10.10.1.12 --------- Controller Manager
||----- 10.10.1.13 --------- Kube-Proxy
||----- 10.10.1.14 --------- Etcd
||                       Kubelet
||
+- 192.168.1.201 - Nodo1
||                       Linux
||                       Crio
||----- 10.10.1.20 --------- Kube-Proxy
||                       Kubelet
+- 192.168.1.202 - Nodo2
||                       Linux
||                       Crio
||----- 10.10.1.15 --------- Kube-Proxy
||                       Kubelet
+- 192.168.1.203 - Nodo3
||                       Linux
||                       Crio
||----- 10.10.1.16 --------- Kube-Proxy
||                       Kubelet
+- 192.168.1.204 - Nodo4
||                       Linux
||                       Crio
||----- 10.10.1.17 --------- Kube-Proxy
||                       Kubelet
 |
 |
 +--------------- VLAN de kubernetes (10.10.0.0/16)

Kubelet: es un servicio (gestionado con systemctl, se instala a hierro) que corre en cada nodo (maestro y worker).
         Su trabajo es dar órdenes al gestor de contenedores (cri-o) para que cree, borre, inicie, pare contenedores.
Adicionalmeente hay que instalar muchos más programas.
Kubernetes no es un único programa... es un conjunto de programas que trabajan juntos para crear y operar el entorno de producción.
El Kubelet es solo uno de esos programas... y el único que instalamos a hierro (een realidad hay otro - kubeadm)
El resto de programas de kubernetes se instalan como CONTENEDORES dentro del propio cluster (en los nodos maestros).
Este conjunto de contenedores que instalamos en los nodos matestros recibe el nombre de PLANO DE CONTROL (Control Plane):
- Una BBDD: etcd
- Un DNS: coreDNS (Un dns para comunicaciones internas del cluster)
- ApiServer: Es la puerta de entrada al cluster. Todo lo que queramos hacer en el cluster, se lo pido al ApiServer
             Nos ofrece servicios RESTful (HTTP/HTTPS) a los que epuedo llamar.
             Cuando trabajemos con el clientee de lineea de comandos (kubectl/oc) o cuando trabaje con la UI web de Kubeernetes/Openshift, lo que estoy haciendo es llamar al ApiServer.
- Scheduler: Programa cuya única misión es decidir en qué nodo (worker) se va a ejecutar cada programa.
- Controller manager: Este es el cerebro de kubernetes... es que gestiona todo el trabajo.. el que toma decisiones..
- Kube-proxy: (Este es especial). Por ahora no os digo lo que hace
             De éste tenemos una copia en cada nodo (maestro o worker)  
             Del resto, tendremos 2 copias (alta disponibilidad) en los nodos maestros

Los gestores de contenedores crean una Red Virtual privada dentro del host. Eso no vale en kubernetes.
Necesitamos una comunicación constantee entre todos lso contenedores que corren dentro del cluster.
Para ello, necesitamos montar una RED VIRTUAL (REAL.. no dentro de un host... sino una que opera sobre una red FISICA) que seea transversal a todos los nodos del cluster.

Hay un montón de plugins de red que hacen eso compatible con kubernetes:
- Flannel
- Calico    <<<<
- Weave
- Cilium
- ...
---

Las BBDD que operan en cluster.. la mayoría, se enfrentan a un problema: BRAIN SPLIT
Pasa lo mismo en elasticSearch, mongo, mariadb en cluster, etcd...

    MariaDB1  dato1                                     <<
        ||
    MariaDB2* dato1                                     <<      BC        <  Cliente
        xx
    MariaDB3  dato2                                     <<


    Dato1 | Dato2

    En este tipo de sistemas, uno dee los nodos toma el role maestro:
    - Asigna identificadores únicos a los datos
    - Determina en qué nodo se guarda cada dato
Para resolver esa situación, en la que algún nodo puede perder la comunicación con los otros nodos, el maestro se decide por consenso entre todos los nodos.. exigiendo mayoría absoluta (más de la mitad de los nodos deben estar de acuerdo en quién es el maestro).

---

# Pod

En kubernetes no se despliegan Contenedores. Trabajamos con PODS.
De hecho, algunos gestores de contenedores (podman = pod manager) ya usan el concepto de pod. EEn docker no existe el concepto de pod.

> Qué es un pod:

Un conjunto de contenedores (al menos 1), que:
- Se despliegan en el mismo host (nodo) =>
   - Comparten configuración de red (misma IPs.. y pueden hablar entre si con localhost)
   - Pueden compartir volúmenes locales (dentro del nodo)
- Comparten el mismo ciclo de vida (se crean, destruyen, arrancan, paran juntos)
- Escalan juntos. Lo que vamos a crear son réplicas del pod... y por ende de todos los contenedores que forman el pod.

---

Para instalar Wordpress... Necesito:
- Apache (php+wordpress)
- MariaDB

> Pregunta: 1 contenedor o 2? SIEMPRE SEPARADOS.

Y da igual... Nunca instalo 2 programas en el mismo CONTENEDOR. Para qué?
Todo programa siempre en su entorno aislado
- Contenedor MariaDB
- Contenedor Apache+php+wordpress

> Pregunta: 1 pod o 2 pods? 

2 pods. En este caso, el escalado manda. No quiero que escalen juntos. NO ES DESEABLE
---

Quiero añadir un tercer programa: Filebeat (para enviar logs del apache a ElasticSearch)

> Pregunta 1: El filebeat lo pongo en su propio contenedor o lo meto en el contenedor del apache?

Siempre separados. Siempre 1 programa por contenedor.
Entonces.. ahora tengo 3 contendores:
- Contenedor MariaDB
- Contenedor Apache+php+wordpress
- Contenedor Filebeat

> Cuántos pods?

- Pod MariaDB (1 contenedor)
- Pod (Apache+php+wordpress) + (filebeat) (2 contenedores)
  Por qué? 
  - [√] Se despliegan en el mismo host (nodo) =>
   - [x] Comparten configuración de red (misma IPs.. y pueden hablar entre si con localhost)
   - [√] Pueden compartir volúmenes locales (dentro del nodo)
  - [~] Comparten el mismo ciclo de vida (se crean, destruyen, arrancan, paran juntos)
      El filebeat va subordinado al apache. Contenedor SIDECAR 
  - [√] Escalan juntos. Lo que vamos a crear son réplicas del pod... y por ende de todos los contenedores que forman el pod.
      Cada Apache que se despliegue, tendrá su filebeat asociado 

---

# Objetos en kubernetes

Hemos dicho que lo único que hacemos en kubernetes es definir objetos en el cluster.
Esos objetos pueden ser de diferentes tipos.
Cada objeto se identifica por:
- Un tipo de objeto que es
- Un nombre único que le asigno

    pod/apache-1
    pod/apache-2
    pod/apache-3
    pod/mariadb
    configmap/apache
    configmap/mariadb

PERO... esto mee temo que no sería suficiente.
> Pregunta... El despliegue (cluster) del mariadb+Wordpress que queremos hacer... cuántas veces queremos hacerlo en nuestro cluster?
 - Wordpress de producción
 - Wordpress de preproducción/test
 - ...

Si cada uno de esos despliegues lleva  nombres diferentes... acabo con varias configuraciones diferentes para el mismo aplicativo (mariadb+wordpress). Esto no nos interesa.

En kubernetes existe el concepto de NAMESPACE: Espacio de nombres.

# Namespace

Es un entorno LOGICO dentro del cluster de kubernetes.

Un cluster es un entorno FISICO (con sus nodos, red, etc)
Dentro de ese cluster puedo tener 500 entorno LOGICOS diferentes (namespaces)
Dentro de cada uno de esos entornos, los NAMES (nombres de los objetos) son únicos.
Lo que realmente identifica un objeto en kubernetes:
- Namespace
- Un tipo de objeto que es
- Un nombre único que le asigno

Los namespaces nos sirven para:
- Crear entornos diferentes de despliegues de una app: producción, preproducción, test, desarrollo, etc
- Separar instalaciones del mismo producto para distintos clientes (multi-tenant): app1-clienteA, app1-clienteB, etc

Veremos que podemos limitar el consumo de recursos (CPU, RAM, etc) por namespace... LimitRange / ResourceQuota
Veremos que cada namespace puede tener sus propios administradores (ServiceAccount, Role, RoleBinding, etc)

En el trabajo del día a día, un administrador de un cluster de kubernetes, cuando hay que hacer un nuevo despliegue, lo que hace es:
- Crear un namespace para ese despliegue
- Crea una cuenta de usuario: ServiceAccount para ese proyecto
- Lee asigna roles (Role, RoleBinding) a esa cuenta de usuario para administrar ese namespace
- Limito el consumo de recursos de ese namespace (ResourceQuota / LimitRange)
- Y entrega esa cuenta de usuario al equipo de desarrollo / operaciones que va a gestionar ese despliegue.
- A partir de ahí, ese equipo solo podrá operar dentro de su namespace (no podrá tocar otros despliegues en otros namespaces)

Por ejemplo:
- Namespace: app1-produccion
    - pod/apache-1
    - pod/mariadb 
    - configmap/apache
    - configmap/mariadb
- Namespace: app1-preproduccion
    - pod/apache-1
    - pod/mariadb 
    - configmap/apache
    - configmap/mariadb

Todo objeto (salvo muy gloriosas excepciones que ya os contaré) se crea siempre dentro de (asociado a) un namespace. Por ejemplo, un namespace no se crea dentro de otro namespace. 
Por ende, los nombres dee los namespaces son únicos en todo el cluster.

Cómo creo un namespace en kubernetes. Lo primero: YO NO CREO EL NAMESPACE.
Yo digo que quiero tener un namespace (lenguaje declarativo.. no imperativo)

---

> Qué tipo de programas puedo poner a funcionar en un contenedor (con docker mismo)?

- BBDD? SI              |
- Servidor web? SI      | Servicio... que queda configurado y corriendo PARA SIEMPRE JAMAS 

- Script?       ETL. SI

Dentro de un contenedor, podemos ejecutar distintos tipos de programas.
Eso en general.
Pero no así en Kubernetes:
Kubernetes da por supuesto que los contenedores de un pod lo que llevan dentro son Servicios (programas que quedan corriendo PARA SIEMPRE JAMAS). Si se me ocurriese la idea de poner un script dentro de un pod de kubernetes... la lio. Kubernetes se desquicia, se vuelve loco.. Lo arranca, y ve que acaba.. y piensa... Esto ha ido mal.. no debería haber acabado. Inmediatamente lo vuelve a arrancar... y si vuelve a acabar... otra vez lo vuelve a arrancar... y así hasta el infinito. En una hora, me puede haber ejecutado aquello 40000 veces.

En cuanto aplico el manifiesto de un pod en kubernetes, Kubernetes entiende que quiero tener en funcionamiento sus contenedores PARA SIEMPRE JAMAS. Y los arranca... y más vale que no acaben.. que se queden corriendo. Sino, Kubernetes va a reiniciarlos una y otra vez hasta que queden arriba.

En Kubernetes no existe el concepto (como en docker) de arrancar un contenedor, para un contenedor. ESO NO EXISTE.
Lo que existe es:
- QUIERO TENER ESTE POD, con sus CONTENEDORES, en mi entorno de producción.
- Y kubernetes su única misión a partir de ahí es: ASEGURARME DE QUE los programas que corren en los contenedores de ese pod ESTÉN ARRIBA PARA SIEMPRE JAMÁS.

---

# Asignación de recursos a contenedores en kubernetes

```yaml
     resources:
        requests:
          memory:   512Mi
          cpu:      500m
        limits:
          memory:   1Gi
          cpu:      2
```

Qué significan esas cosas?
De entrada.. hablemos de las unidades:
- Memory: 1 Gigabyte? 1 GibyByte?
    1 GB = 1000 MB
    1 MB = 1000 KB
    1 KB = 1000 Bytes (Desde hace más de 25 años) Antiguamente eran 1024. Pero eso cambió en el 1999 (Según norma ISO)
    En ese momento se crearon un nuevo juego de medias: Bibytes
     1 GiB = 1024 MiB
     1 MiB = 1024 KiB      Es decir, que hoy en día, los bibytes son lo que para nosotros han sdo toda la vida los Kilo.. Mega..bytes.
     1 KiB = 1024 Bytes
- CPU: Puedo poner número enteros (1, 2, 3, etc) o en millicores (m)
   - 1 CPU = 1 núcleo completo de CPU
   - 500m = 0.5 CPU = medio núcleo de CPU
  No son CPUs (Ni físicas ni cirtuales: hyper-threading) que se asignan al contenedor.
  Es el equivalente el tiempo de uso que permito que use el contenedor en un núcleo (virtual: hyperthreading) de CPU.

    Si pongo 1, significa que el contenedor puede usar hasta el equivalente al 100% de un núcleo de CPU constantemente en el tiempo...
        Puede ser que mi programa esté usando:
            - Core1 al 100% durante
            - Core1 al 50% durante
              + Core2 al 25% durante
              + Core3 al 25% durante
    Si pongo 500m, significa que el contenedor puede usar hasta el equivalente al 50% de un núcleo de CPU constantemente en el tiempo.
        Puede ser que mi programa esté usando:
            - Core1 al 50% durante
            - Core1 al 25% durante
              + Core2 al 15% durante
              + Core3 al 10% durante

# Requests y limits

Request, lo que el contenedor solicita que se le garantice.
Limits, lo que el contenedor pide, que si hay opción, se le permita usar.

 > Cluster

            CAPACIDAD    |    USO    | SIN USAR    | COMPROMETIDO | SIN COMPROMETER
            CPU   RAM.   | CPU   RAM | CPU   RAM   | CPU     RAM  | CPU    RAM
   ----------------------+-----------+-------------+--------------+-----------------------------------------------------
   Nodo1    10     20    |  4    10  |  6     10   | 9       20   |  1      0
    pod-apache-1         |  3     7  |  1      3   | 4       10 
    pod-mariadb          |  1     3  |  4      7   | 5       10
   ----------------------+-----------+----------------------------------------------------------------------------------
   Nodo2    10     20    |  0     0  |
    pod-apache-1         |  3     7  |  1      3   | 4       10 
   ----------------------+-----------+----------------------------------------------------------------------------------


   Desplegar:    REQUESTS          LIMITS
                CPU   RAM        CPU   RAM
    pod-apache    4    10          6    15
    pod-mariadb   5    10          6    15

El scheduler solo se fija en los REQUESTS para decidir si puede o no desplegar un pod en un nodo.
El scheduler no se fija en lo que esté o no en USO para decir si puede o no desplegar un pod en un nodo.
Hay otro concepto: LO QUE EL NODO TIENE COMPROMETIDO

Esto es en cuanto al Scheduler.
Veremos, que el scheduler, además de tener en cuenta los request de un pod y lo no comprometido de un nodo, también tiene en cuenta otras cosas (afinidades, anti-afinidades, taints, tolerations, etc)... de eso ya hablaremos.

Pero.. que pasa luego en ejecución... una vez ya ha sido planificado?


            CAPACIDAD    |    USO    | SIN USAR    |
            CPU   RAM.   | CPU   RAM | CPU   RAM   |
   ----------------------+-----------+-------------+
   Nodo1    10     20    | 10    20  |  0      5   |
    pod-apache-1         |  0     0  |  0      0   |
    pod-mariadb          |  5    10  |  0      0   |
   ----------------------+-----------+--------------

   A un proceso le puedo encolar las peticiones a la CPU (que espere su turno)... y hacer que vaya más lento.
   Pero a un proceso no le puedo quitar un cacho de RAM que ya tiene asignada.... dejo el proceso totalmente inestable.
    ESTE CASO ES DISTINTO al de la CPU.
    Con la CPU no hay problema con poner un valor muy alto. Si me paso, el kernel de linux se encarga de gestionar la cola de procesos y repartir el tiempo de CPU entre todos los procesos que lo pidan de acuerdo a lo garantizado.
    Pero con la RAM, si me paso en los limits... el kernel de linux no puede hacer nada.
    Solución: Kubernetes hace un kill -9 al apache y punto.. que se joda por listo! TAL CUAL!

Es decir:
- Requests: Lo que considero realmente necesario para mi programa
- Limits:
  - CPU: Dame todo!
  - RAM... LO MISMO QUE EN REQUESTS! Esa es la recomendación (a no ser que tenga muy claro lo que estoy haciendo!)
    Esto no es nuevo... Weblogic -> JVM
        java -Xms512m -Xmx512m... Cuál es la recomendación en JAVA? Que Xms y Xmx sean iguales!

Hay muchos escenarios donde jugamos mucho con esto.
En un cluster acabamos con miles de programas.
- Quiero un programa para proveer certificados SSL (cert-manager)
- Quiero un programa para provisionar volúmenes de almacenamiento (rook-ceph)

Cuándo se ejecutan esos programas?
- CertManager? 1 vez cada 3 meses... durante 2 segundos... Y voy a tenerle reservados 500m de CPU y 1Gi de RAM? NO JODAS !
- Rook-Ceph? Cuando despliego por primera vez una app.. durante 5 segundos... Y voy a tenerle reservados 1 CPU y 2 Gi de RAM? NO JODAS!

Me quedo sin cluster!

A este tipo de programas, usualmente le configuro un request de CPU : 50m y RAM: 10Mi y un limit: de CPU: 2 y RAM: 1Gi


# Quien define los requests y limits de un contenedor dentro de un pod?

Quién sabe cuanta CPU y RAM necesita un programa?
- Administrador de sistemas? NO
- Desarrollador? NO
No lo sabe nadie al principio... podemos tener estimaciones más o menos precisas... pero nadie lo sabe a ciencia cierta.
La única forma de saberlo es MONITORIZACION! Con el tiempo vamos ajustando.
Lo bueno y con lo que hay que tener un huevo de cuidado es que no me es importante los valores absolutos. Lo UNICO que me tengo que preocupar es de conseguir un buen RATIO entre CPU y RAM.

A más CPU, más RAM necesito.

Estoy con un microservicio. Más CPU, es que más peticiones por segundo voy a poder atender.
Pero para atender a cada petición, voy a necesitar un cacho de RAM.

Si subo CPU, tengo que subir RAM.
Lo que me puede pasar es que configure un mal ratio:
    CPU: 3
    RAM: 10Gi

    Y que con la cantidad de operaciones que es capaz de procesar con 3 CPUs, solo necesite 5Gi de RAM.
    Nunca va a necesitar más RAM, porque con 3 CPUs no puede procesar más operaciones por segundo.
    Y resulta que tengo 5 Gi de RAM reservadas que no voy a usar nunca.

    En kubernetes tiramos de escalado HORIZONTAL!
    Si los 3 cores no son suficientes, añado otro pod... otros 3 cores a procesar.
    Sigo apretao.. otro pod.. con otros 3 cores... y así hasta que el cluster aguante o llegue al límite configurado para mi namespace (ResourceQuota)
    Ahora .. si no teengo bien ajustado el ratio CPU/RAM... estaré reservando RAM a kilos quee no voy a usar nunca.... y escalando horizontalmente, es decir x10, x20, x30... a cantidades de memoria RAM que no voy a usar nunca.


## Limitación de recursos en namespace:

Se usa el objeto ResourceQuota

```yaml
kind: ResourceQuota
apiVersion: v1

metadata:   
    name:      limitar-recursos-despliegue-wordpress
    namespace: app1-produccion

spec:
    hard:
        requests.cpu:      "20"
        requests.memory:   40Gi
        limits.cpu:        "30"
        limits.memory:     60Gi
        pods:              "10"
        persistentvolumeclaims: "5"
        rapidito.storageclassclaims.storage.k8s.io/fast: "5Gi"

```
No es obligatorio. Si no se define... ancha es castilla!
Siempre cuando un administrador de un cluster crea un namespace para un equipo de desarrollo/operaciones, le asocia inmediatamente un ResourceQuota para limitar el consumo de recursos de ese namespace.

Hay un segundo objeto (para un control más fino) que es LimitRange.

```yaml
kind: LimitRange
apiVersion: v1  
metadata:   
    name:      limitar-recursos-por-pod
    namespace: app1-produccion
spec:
    limits:
    - type:       Container
      max:
        cpu:      "4"
        memory:   8Gi   
```

No te dejo que en pod dentro de ese namespace, haya un contenedor que pida más de 4 CPUs o 8Gi de RAM.
Me cuesta mucho encajar pods tan grandes en los nodos del cluster.
Yo, administrador del cluster, lo que me interesa es que tus pods sean pequeños.. Cuanto más pequeños, mejor los encajo en los nodos del cluster... y mejor aprovecho los recursos del cluster.

Este .. no le damos tanta importancia al menos tanta como al ResourceQuota
El resourceQuota debe ir siempre.
Si quiero hacer un control más fino, también uso LimitRange

---

# Volumenes en Kubernetes

## Para que sirve un volumen?

- Persistencia de datos tras el borrado del contenedor/pod
- Inyectar configuraciones a los contenedores
- Compartir datos entre contenedores

En kubernetes hay muchos tipos de volúmenes... y cada uno va orientado a un uso diferente.

- Compartir datos entre contenedores de un mismo pod: 
        emptyDir: Esto crea un volumen temporal a nivel local! <<<<.         Apache - Filebeat (leyendo los logs)
        Inicialmente vacío... y con persistencia temporal por defecto a nivel de disco duro.
         Cuando creamos un volumen de este tipo, kubernetes crea en la máquina donde se despliega el pod, un directorio temporal (normalmente en /var/lib/kubelet/pods/ID_POD/volumes/kubernetes.io~empty-dir/NOMBRE_VOLUMEN)
         Esa carpeta inicialmente está vacía.
         Todos lo contenedores del pod pueden leer y escribir en esa carpeta.
         Cuando el pod es eliminado, esa carpeta se borra y se pierde toda la información que había en ella.

         Este tipo de volumen puede configurarse para tener persistencia en RAM (tmpfs) o en disco duro (por defecto).

             Apache -> access.log <- Filebeat -> ElasticSearch (persistencia real de la buena)
                          ^
                     Guardarlo en RAM (Usar un trozo de la RAM como si fuera una carpeta)

             Ese fichero lo quiero persistir yo a nivel del host? NO
             La persisteencia a los datos del log la hace el gestor de logs (ElasticSearch, Splunk, etc) que recibe los logs desde Filebeat.

                En un caso como este, podemos configurar eel apache con un log rotado entre 2 ficheros de 50Kbs. Limita el consumo de RAM a 100Kbs para este tema (despreciable) y me da un rendimiento brutal (no escribo nada en disco duro)

- Inyectar configuraciones a los contenedores:
        configMap:   Inyecto ficheros de configuración en el contenedor
        secret:      Inyecto ficheros sensibles (contraseñas, certificados, etc) en el contenedor

- Persistencia real de datos tras el borrado del contenedor/pod:
        Hay cientos de tipos de volumenes.
        Están lo que kubernetes ofrece por defecto:  nfs, iscsi, glusterfs, cephfs, etc  
        Cada fabricante de cabinas de almacenamiento (NetApp, DellEMC, HPE, etc) ofrece su propio plugin de volumenes para kubernetes.

Los volumenes en Kubernetes, a diferencia de en docker, no se crean a nivel del contenedor, sino a nivel del POD.

---

# Configmaps y Secrets

Tienen 2 usos:
- Dar valores para variables de entorno (env) a los contenedores
- Inyectar ficheros de configuración en el filesystem del contenedor 
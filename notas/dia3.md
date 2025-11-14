
# Volumenes persistentes

```yaml
---
kind:             PersistentVolume
apiVersion:       v1
metadata:
  name:           volumen1
spec:
  capacity:
      storage:        20Gi
  storageClassName:   rapidito-redundante
  accessModes:
    - ReadWriteOnce
  nfs:
    server:   mi-nfs-server.mi-dominio.local
    path:     /ruta/del/volumen
---
kind:             PersistentVolumeClaim
apiVersion:       v1
metadata:
  name:           pvc-datos-mariadb
spec:
  resources:
    requests:
      storage:    10Gi
  storageClassName:   rapidito-redundante
  accessModes:
    - ReadWriteOnce     
```
Damos estos documentos dee alta en kubernetes. Uno lo da de alta el administrador del cluster (PV) y otro lo da de alta el usuario/desarrollador (PVC).
Qué pasa entonces.. entra Kubernetes, que es el TINDER de los volúmenes. El que hace MATCH.
Mira lo que pide el desarrollador en la PVC y busca un PV que satisfaga esos requisitos.

            pvc-datos-mariadb <---> volumen1 (se vinculan)

        > Pregunta: Se le dan los 10Gbs que pidió o los 20Gbs que tiene el PV? Los 20!

Si una PV se crea y no hay todavía PVCs con los que pueda hacer match... se queda Disponible en el cluster.
Si una PVC se crea y no hay PVs disponibles que satisfagan sus requisitos... se queda en estado Pending.

Todo esto dijimos que es para automatizar!

Antiguamente, para lidiar con este problema lo que hacíamos desde administración del cluster era crear 50 volúmenes PVs, de distintos tamaños.. y ponerlos ahí en el cluster. De esta forma, cuando un desarrollador lanzaba un PVC, kubernetes podía hacer match con alguno de los PVs ya existentes.

Esta solución era un poco RUINA!

Hoy en día, el administrador lo que hace es instalar un PROVISIONADOR DINÁMICO de volúmenes persistentes.

## Qué es esto?

Es un programa que instalo en el cluster de kubernetes, y ese programa:
1. Está escuchando todas las PVCs que se crean en el cluster
2. Cuando detecta que se ha creado una PVC nueva, lee sus datos (tamaño) y mira si el tipo de almacenamiento que pide (storageClassName) es un tipo que él puede gestionar
      EMC2: Cabina Discos Rotacionales  <- storageClassName: lento                          <- Provisionador Dinámico de EMC2
      Huawei: Almacenamiento SSD        <- storageClassName: rapidito                       <- Provisionador Dinámico de Huawei
      NAS: Servidor NFS                 <- storageClassName: lento(AccessMode: Compartido)  <- Provisionador Dinámico de NFS

      Al menos, necesito un provisionador para volúmenes de bloques (RWO) y otro para volúmenes compartidos (RWX) (orientado a fichero, orientado a objetos S3 - Minio)
3. Va al backend y crea AUTOMATICAMENTE EL VOLUMEN.
4. Registran ese volumen recién creado en kubernetes como un PV, marcan esa PV como ASIGNADO (Bound) a la PVC que lo ha solicitado.

---

# Instalar el provisionador:

Se despliegan programas en el cluster: Normalmente con HELM

En casos más avanzados, además hay que:
- Instalar clientes específicos del provisionador
- Configurar acceso a los backends de almacenamiento (credenciales, IPs, rutas, etc)

Además, hay otras cosas a tener en cuenta. Por ejemplo, si vamos a trabajar con nfs, necesito en todos los NODOS del cluster tener instalado el cliente nfs (paquete nfs-common en debian/ubuntu)

Si por el contrario (o adicionalmente) voy a trabajar con iscsi, necesito tener instalado el cliente iscsi en todos los nodos del cluster (paquete open-iscsi en debian/ubuntu)

Hemos hablado de escalado horizontal automático de pods. Muchas veces no quiero escalar solamente los pods... también quiero escalar los nodos del cluster. Tengo un cluster con 5 nodos de trabajo... pero.. en momentos quiero poder tener hasta 15 nodos, para poder albergar más pods. Y quiero que se haga DINAMICAMENTE y AUTOMATICAMENTE:
- Cloud: EKS, Openshift-> Cloud... llevan programas y Tipos de objetos (CRDs) adicionales en Kubernetes para gestionar el escalado automático de nodos, donde en automático se crean nuevos nodos(o eliminar) en el cloud (AWS, Azure, GCP) y se añaden al cluster de kubernetes
- On prem: 
  - VMWare: Tanzu
  Tanzu (la distro de kubernetes de VMWare) tiene un componente llamado VMWare Cloud Director que permite gestionar el escalado automático de nodos en un cluster on prem.
  - Karbon: La distro de kubernetes de Nutanix. Permite gestionar el escalado automático de nodos en un cluster on prem.
Pero claro... luego hay que configurar esos nodos para por ejemplo que soporten cierto tipo de almacenamiento: clientes de nfs, iscsi, etc. Eso es otra cosa que me regala : VMWare, Nutanix, etc.

---

# Plantillas de despliegue en kubernetes

Se llaman Charts, y los gestiona una herramienta llamada HELM.
Esta es la forma normal de instalar cualquier software COMERCIAL!

Las descargo de artifacthub.io ~ dockerhub.com 
                      ^             ^
                      ^            imágenes de contenedor
                Plantilla de despliegue

Las configuro con un archivo values.yaml que me dan

Y en automático HELM genera el archivo de manifestos yaml para kubernetes y lo despliega.

---

# Comunicaciones en Kubernetes

    192.168.1.50:80 ->
        192.168.1.101:30080
        192.168.1.102:30080
        192.168.1.103:30080
        192.168.1.201:30080
        192.168.1.202:30080
        192.168.1.203:30080
        192.168.1.204:30080            app1 -> 192.168.1.50        https://app1:80
         |                                 |                           |
       Balanceador                     DNS corporativo               MenchuPC
         |                                 |                            |
      192.168.1.50                    192.168.1.51                  192.168.27.182
         |                                 |                           |
+--------+---------------------------------+---------------------------+---- red de la empresa (192.168.0.0/16)
|
+- 192.168.1.101 - NodoMaestro1
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.101:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.1.1 ---------- POD: CoreDNS (1 contenedor)
||                                  mariadb-service     <- 10.10.3.100
||                                  wordpress-service   <- 10.10.3.101
||                                  nginx-ingress       <- 10.10.3.102
||----- 10.10.1.2 ---------- ApiServer
||----- 10.10.1.3 ---------- Scheduler
||----- 10.10.1.4 ---------- Etcd
||----- 10.10.1.5 ---------- Kube-Proxy <<< Este es el que va editando las reglas de netfilter en cada nodo
||                       Kubelet
+- 192.168.1.102 - NodoMaestro2
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.102:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.1.6 ---------- POD: ApiServer (1 contenedor)
||----- 10.10.1.7 ---------- Etcd
||----- 10.10.1.8 ---------- Controller Manager
||----- 10.10.1.9 ---------- Kube-Proxy
||                       Kubelet
+- 192.168.1.103 - NodoMaestro3
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.103:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.1.10 --------- Scheduler
||----- 10.10.1.11 --------- CoreDNS
||                                  mariadb-service     <- 10.10.3.100
||                                  wordpress-service   <- 10.10.3.101
||                                  nginx-ingress       <- 10.10.3.102
||----- 10.10.1.12 --------- Controller Manager
||----- 10.10.1.13 --------- Kube-Proxy
||----- 10.10.1.14 --------- Etcd
||                       Kubelet
||
+- 192.168.1.201 - Nodo1
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.201:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.2.103 -------- POD Wordpress
||                                 Contenedor Wordpress
||                                      Proceso Apache: 10.10.2.103:80
||                                      /etc/wordpress/wp-config.php <- DATABASE_HOST=mariadb-service:3307
||----- 10.10.1.18 --------- Kube-Proxy
||                       Kubelet
+- 192.168.1.202 - Nodo2
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.202:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.1.15 --------- Kube-Proxy
||----- 10.10.2.101 -------- POD Wordpress
||                                 Contenedor Wordpress
||                                      Proceso Apache: 10.10.2.101:80
||                                      /etc/wordpress/wp-config.php <- DATABASE_HOST=mariadb-service:3307
||                       Kubelet
+- 192.168.1.203 - Nodo3
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.203:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.2.104 -------- POD IngressController
||                                 Contenedor Nginx
||                                      Proceso nginx: 10.10.2.104:80
||                                           nginx.conf <- Reglas de proxy reverso:
||                                             app1.com -> wordpress-service:80   (INGRESS) 
||----- 10.10.1.16 --------- Kube-Proxy
||                       Kubelet
+- 192.168.1.204 - Nodo4
||                       Linux
||                             Netfilter
||                                 10.10.3.100:3307 -> 10.10.2.102:3306
||                                 10.10.3.101:80   -> 10.10.2.101:80 | 10.10.2.103:80
||                                 192.168.1.204:30080 -> 10.10.3.102:80
||                                 10.10.3.102:80   ->  10.10.2.104:80
||                       Crio
||----- 10.10.2.102 -------- POD MariaDB
||                                 Contenedor MariaDB
||                                      Proceso MariaDB: 10.10.2.102:3306
||----- 10.10.1.17 --------- Kube-Proxy
||                       Kubelet
 |
 |
 +--------------- VLAN de kubernetes (10.10.0.0/16)

 Pod MariaDB.

SERVICE 
    CLUSTERIP = IP FIJA de BALANCEO en la Red virtual del cluster + Entrada en DNS interno de kubernetes (Comunicaciones internas)

        servicio mariadb: mariadb-service       <- 10.10.3.100
                                                puerto 3307 -> Pod mariadb: 3306

    NODEPORT = CLUSTERIP + NAT en todos los nodos del cluster (puerto>30000) (Comunicaciones externas)

        servicio wordpress: wordpress-service   <- 10.10.3.101
                                                puerto 80 -> 1 de los Pod wordpress: 80
                                                + puerto 30080 -> 10.10.3.101:80  (NAT en todos los nodos)

    LOADBALANCER = NODEPORT + Gestión automatizada de un balanceador de carga externo (siempre que sea compatible con kubernetes) 
    Si trabajo con un cloud, cualquier cloud me "regala" (€€€€) un balanceador de carga compatible con kubernetes.
    EKS, AKS, GKE, Openshift->Cloud
    El problema es si quiero montar un cluster de kubernetes on prem... necesito un balanceador de carga compatible con kubernetes.
        En este caso, esa instalación me la como yo <- MetalLb


ROUTE (Openshift) = LOADBALANCER + Gestión automatizada de certificados SSL + Gestión automatizada de nombres DNS públicos
    DNS (Wildcard) + nombre de cluster:          mi-cluster.com
    Nombre de la aplicación:                app1.mi-cluster.com
                                            app2.mi-cluster.com


> En un cluster Tipo (ESTANDAR) Que nos encontremos por ahí de kubernetes..


    Servicios           %
    --------------------------------
    ClusterIP           TODOS - 1        (Comunicaciones internas)
    NodePort            0
    LoadBalancer        1                (Comunicaciones externas) - Proxy Reverso!


Ese proxy reverso, en el mundo de kubernetes se denomina: INGRESS CONTROLLER
Kubernetes por defecto no trae ningún Ingress Controller instalado.
Es algo que yo tengo que instalar... y hay un huevo de opciones disponibles.

Un ingress controller son 2 cosas:
- Proxy reverso (nginx, haproxy, traefik, etc)
  + Un programa que configura ese proxy reverso para que sepa redirigir las peticiones a los servicios adecuados


Las reglas que configuramos en un proxy reverso llevan siempre los mismos datos:
- Nombre DNS al que va dirigida la petición                 app1.miempresa.com
- Puerto al que va dirigida la petición                     80, 443  
- URL de Backend al que se redirige la petición             http://mi-servidor17.mi-empresa.internal:8080/app1
La sintaxis concreta depende del proxy reverso que estemos usando.

Kubernetes me ofrece una sintaxis UNIFICADA para definir esas reglas de proxy reverso, donde meto esos datos: INGRESS
Un ingress en kubernetes es una regla de proxy reverso.

Cuando instalo un Ingress Controller en kubernetes:
- Proxy reverso (nginx, haproxy, traefik, apache...)
  + programa que en base a los ingress que vayamos dando de alta en kubernetes, es capaz de configurar el proxy reverso  concreto que usemos para que sepa redirigir las peticiones a los servicios adecuados. 

```yaml

kind:           Ingress
apiVersion:     networking.k8s.io/v1

metadata: 
  name:         ingress-wordpress

spec:
    rules:
    - host:     app1.mi-cluster.com
      http:
        paths: 
         - path:     /
           pathType: Prefix
           backend:
             service:
               name:    wordpress-service
               port:
                 number: 80

```

El ingress controller me instala un nginx y además traduce esa regla al formato concreto de nginx.

```
server {
    listen 80;
    server_name app1.mi-cluster.com;

    location / {
        proxy_pass http://wordpress-service:80;
    }
}
```
Si mañana cambio el proxy reverso de nginx a haproxy... tendré un nuevo ingress controller que traducirá esa misma regla al formato concreto de haproxy:

```
frontend http-in
    bind *:80
    acl host_app1 hdr(host) -i app1.mi-cluster.com
    use_backend wordpress-backend if host_app1
backend wordpress-backend
    server wordpress-service wordpress-service:80
```
---

# NetFilter

Es un componente del kernel de linux por el que pasa CUALQUIER PAQUETE DE RED que entra o sale de una máquina.

IPTABLES (es solo una forma de dar reglas a netfilter)

---

> Pregunta: Cuántos pods vamos a crear nosotros en un cluster de kubernetes?

CERO!

Yo no quiero crear pods en kubernetes. Quiero que kubernetes cree los pods.
Puedo crearlos? SI
Quiero crear yo los pods? NO

Si tengo el wordpress... y tiene 1 pod (que yo he creado)... y quiero que ahora haya 4 pods... a quién le toca crear esos otros 3 pods? A MI... Y esto no es lo que queremos!

Entonces... como funciona este tinglao? Lo que vamos a hacer nosotros es darle a kubernetes una plantilla de pod.

    - Deployment    = Plantilla de Pod + Número de réplicas Iniciales
        Apps web, microservicios ... todo eso son Deployment
    
    - StatefulSet   = Plantilla de Pod + Número de réplicas Iniciales + Al menos una Plantilla de Petición de Volumen Persistente
            Todo lo que sean sistemas de almacenamiento de datos en cluster: BBDD, Sistemas de mensajería (KAFKA, RabbitMQ), Indexadores (ElasticSearch) , etc ... deben ser creados con StatefulSet

    - DaemonSet. Es raro que lo usemos nosotros para despliegues. 
                    = Plantilla de pod de la que no damos un número de réplicas.
                      Ya que kubernetes se encarga de crear 1 pod en cada nodo del cluster.
                      Se reserva para: Herramientas de más bajo nivel en eel cluster:
                      - kube-proxy
                      - agentes de monitorización
                      - herramienta que instale clientes de nfs, iscsi, etc en cada nodo
                      - configurar el max_map_count en cada nodo (para elasticsearch)

---

- Cluster: Activo x Activo (MariaDB Galera, Oracle RAC)
           
           Activo x Pasivo <- En muchos casos es suficiente
                              Es una putada! Porque tengo un entorno continuamente parado!

           Esto en kubernetes es genial!
           Tengo 1 pod... y si se cae, no tengo otro PRECREADO... se crea sobre la marcha.
           Cuánto cuesta arrancar un contenedor y crearlo? Segundos!

           Hay un tiempo de indisponibilidad... mientras arranca.. pero es mínimo... Igual que si tengo un Activo x Pasivo tradicional.

           El tiempo aquí que manda no es el de arrancar el sistema... es tiempo mínimo que configuro para el salto.

---

# Contenedores que corre scripts

En los pods, los contenedores NO pueden correr scripts. Kubernetes obliga a que los contenedores corran PROCESOS que se queden en primer plano indefinidamente. Si se paran... kubernetes considera que el contenedor ha fallado y lo reinicia (aqunque haya devuelto un código de salida 0)

Para poder ejeecuatr scripts o etl, backups de bbdd... hay otro objeto en kubernetes: JOBS

Un Job es como un Pod... pero mientras que en los pods Kubernetes exige que el contenedor corra un proceso en primer plano indefinidamente, en los Jobs kubernetes exige que el contenedor corra un proceso que termine en algún momento (con código de salida 0 o distinto de 0)... Si no termina, Kubernetes considera que el Job ha fallado y lo reinicia.

```

kind :           Job
apiVersion:     batch/v1
metadata:
  name:         job-backup-mariadb
spec:
  template:
    spec:
      containers:
      - name:     contenedor-backup-mariadb
        image:    mi-imagen-backup-mariadb:1.0
        imagePullPolicy:   IfNotPresent
        env:
          - name:   MARIA_DB_HOST
            value:  servicio-mariadb:3307
          - name:   MARIA_DB_USER   
            valueFrom:
              secretKeyRef:
                name:   secret-credenciales-mariadb
                key:    usuario
          - name:   MARIA_DB_PASSWORD
            valueFrom:
              secretKeyRef:
                name:   secret-credenciales-mariadb
                key:    password
      restartPolicy:   OnFailure
```

Hay veces (CASI SIEMPRE) que queremos que estos JOBS se ejecuten en momentos concretos del tiempo. Para eso tenemos otro objeto en kubernetes: CRONJOBS = Plantilla de Job + Programación tipo CRON

```yaml
kind :           CronJob
apiVersion:     batch/v1
metadata:
  name:         cronjob-backup-mariadb
spec:
  schedule:     "0 2 * * *"   # Todos los días a las 2 de la mañana
  jobTemplate:
    spec:
      template:
        spec:         
        # ...
```

---

> Cómo sabe una herramienta tipo docker si el contenedor (su proceso principal) ha fallado o no?

Este tipo de herramientas lo que haceen es mirar si el proceso 1 (de dentro del contenedor) sigue vivo o no.
Y si acaba con código de salida 0 o distinto de 0.

Y es lo mismo que mira kubernetes por defecto.

> Es eso suficiente? NO

Weblogic... configuro un executor pool (pul de hilos de ejecución) con 50 hilos... y se empiezan a quedar un monton de hilos bloqueados... StuckThreads... y al final el servidor de aplicaciones se queda sin hilos disponibles para atender peticiones... pero el proceso 1 sigue vivo!

Qué nos ofrece kunernetes para monitorizar mejor el estado de salud de los contenedores?
Podemos configurar 3 tipos de pruebas diferentes a nivel de cada contenedor en un pod:
- Startup Probe: Son pruebas que nos permiten saber si el proceso ya arrancó o aún no... si sigue arrancando.
                 Las configuro con un delay... con retries... maxFailures... etc
                 Hay procesos que tardan en arrancar... y no quiero que kubernetes interfiera con ese proceso.
                 NextCloud (Google Drive, Calendar, Maps, GMail, Google Docs, etc).
                 En los upgrades, se puede llevar 20 minutos arrancando:
                    - Delay 5 segundos
                    - Un check cada 10 segundos
                    - Si fallan 300 chequeeos seguidos (50 minutos) dalo por fallido el arranque.
                En caso que una prueba de arranque falle, kubernetes considera que el contenedor ha fallado y lo reinicia. 
- Liveness Probe: Una vez que la prueba de arranque se supera...y que el programa ya ha arrancado, comienzan a ejecutarse estas pruebas.
                  Si un programa no responde a estas pruebas, kubernetes considera que el contenedor ha fallado y lo reinicia. 
- Readiness Probe: Si la prueba de vida da positivo, se comienzan a ejecutar estas pruebas, en paralelo con las de vida.
                   Si esta falla, kubernetes saca el pod de balanceo en el servicio asociado.
                   Si se supera es cuando se mete el pod en el balanceo del servicio.
                   Se entiende que el pod está ready cuando todos sus contenedores están ready.
                   Y se entiende que un contenedor está ready, cuando está listo para prestar servicio a usuarios finales.

En cualquiera de esos tipos de pruebas, puedo pedir a kubernetes que:
- Mire un endpoint HTTP/HTTPS (código 200-299 = OK, otro código = FAIL)
- Mire un puerto TCP (si responde = OK, si no responde = FAIL)
- Ejecute un comando (si devuelve código 0 = OK, si devuelve código distinto de 0 = FAIL)

> Ejemplo: 

Arranco Oracle Corporativo... ETA:  5-10 minutos ... durante ese tiempo no quiero reinicios... Si veo que en 15-20 minutos no arranca... ya avisa, probamos a reiniciar!

Ya arrancó... y está lista para prestar servicio. Abrir conexiones y que las apps empiecen a usarla y a lanzar consultas SQL....
Posiblemente quiera monitorizarla cada 5-10 segundos... Si a las 6-10 pruebas (1 minutos/2 minutos) consecutivas no responde... la saco del balanceo... pero... la dejo funcionando... porque ... y si está haciendo un backup?
  
Prueba de vida... el oracle está operativo... y funcionando... aunque puede no estar listo para atender peticiones (esta haciendo un backup) 10 minutos. Si en 10 min no contesta.. reinicio!

# LOGS

Los gestores de contenedores nos permiten ver los "logs" de los procesos que corren en los contenedores:

$ docker logs <container_id>
$ kubectl logs <pod_name> -c <container_name>

Qué se entiende por los logs de un contenedor?
- El proceso 1 que corre dentro de un contenedor debe ejecutarse en primer plano: Bloqueante.
Se entiende por log de un contenedor a la salida ESTANDAR y la salida de ERROR ESTANDAR de ese proceso 1.
---
---

ElasticSearch:
- En ElasticSearch tenemos muchos tipos de nodos: Maestros(3), Data, ML, Coordinadores, Ingest, etc

       MAESTRO 1              MAESTRO 2                 MAESTRO 3

        DATA 1                 DATA 2                   DATA 3          DATA 4


        COORDINADOR 1:9200    COORDINADOR 2:9200
           ^^^^^                 ^^^^^

Cuántos servicios necesito (de kubernetes) para que todo esto funcione correctamente?

ES Habla en el 9200... internamente por el 9300
Las comunicaciones internas se hacen de nodo a nodo por el 9300 ... para esto no hace falta servicio
Si lo querremos para presetar servicio al público (APIs REST, Kibana, etc) por el 9200

Monte un servicio que ataque a los nodos coordinadores por el 9200

Pero...hay que formar el cluster. En el fichero de configuración de ES (el que tiene cada nodo) hay que poner las IPs de los nodos a los que cada nodo se debe presentar, para dar lugar al cluster (UNICAST).

    Arranca un nodo: Maestro1 
    Arranca un nodo: Maestro2 (conf. DISCOVERY_HOSTS = IP_MAESTRO1)
    Arranca un nodo: Maestro3 (conf. DISCOVERY_HOSTS = IP_MAESTRO1, MAESTRO2)
    Arranca un nodo: Data3    (conf. DISCOVERY_HOSTS = IP_MAESTRO1, IP_MAESTRO2)
    
    Data1, Data2... DataN, Coordinador1, Coordinador2... 
      Quiero un fichero de configuración UNICO PARA TODOS! DISCOVERY_HOSTS = maestros-service
    > Me serviría esa configuración para los nodos maestros? DISCOVERY_HOSTS = maestros-service
      Y me temo que no!
          Un servicio para cada uno:       DISCOVERY_HOSTS = maestro1.maestros-service, 
                                                             maestro2.maestros-service, 
                                                             maestro3.maestros-service


    Montaria en Kubernetes un servicio para los nodos maestros: maestros-service

Eso si sería suficiente. Una vez que 2 nodos se presenta mutuamente, cada uno de ellos, preseenta al resto de nodos del cluster con los que ya está hablando.


              MAESTRO1
           ^             ^
        MAESTRO2        MAESTRO3


---
# Planificación del cluster (influir al scheduler). Afinidades, Anti-Afinidades, Taints & Tolerations

El Scheduler de kubernetes es el componente que decide en qué nodo del cluster se va a ejecutar cada pod.
Pero, hay ocasiones en las que me interesa influir en esa decisión? SI, por ejemplo:
- Nodos diferentes: 
  - CPUs más rápidas
  - GPUs (no en todo nodo del cluster tendré GPUs) Modelos AI.

---

- Afinidad a nivel de nodo
  Quiero que este pod vaya en un nodo con GPU 
  Aquí hay 3 opciones de sintaxis que ofrece kubernetes. (NOTA: Realmente 2)

    ```yaml

     #spec del pod:
     #nodeName: nodo-con-gpu-01 # Esto es RUINA. Si se cae el nodo, me quedo sin pod.
     nodeSelector:
       gpu: "true"              # Esto es lo más sencillo. Pero no es muy flexible. Pero en la mayoría de casos es suficiente.
       # Lo que podemos hacer es poner etiquetas en los nodos del cluster:
       # kubectl label node nodo-con-gpu-01 nodo-con-gpu-02 nodo-con-gpu-03 gpu=true
     affinity:
         nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                    nodeSelectorTerms:
                    - matchExpressions:
                        - key: cpu-speed
                          operator: Gt # In, NotIn, Exists, DoesNotExist, Gt, Lt
                          values:
                            - "3000"   # MHz
              preferredDuringSchedulingIgnoredDuringExecution:
                - weight: 1
                    preference:
                    matchExpressions:
                    - key: cpu-speed
                        operator: Gt
                        values:
                        - "2500"
    ```

- Afinidad a nivel de pod
  Quiero que este pod vaya donde ya hay otro pod de tal tipo
  Prefiero el microservicio 1 vaya donde ya está el microservicio 2

- *Antiafinidad a nivel de pod* Se usa siempre
  Prefiero que este pod no vaya donde ya hay otro pod del mismo tipo

```yaml
    #spec del pod: con etiqueta app: wordpress
    affinity:
        podAntiAffinity: # podAffinity
            #requiredDuringSchedulingIgnoredDuringExecution:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
            podAffinityTerm:
                labelSelector:
                    matchExpressions:
                    - key: app
                        operator: In
                        values:
                        - wordpress
                    topologyKey: kubernetes.io/hostname
    ```

Siempre me pasa que no quiero que una app se instale donde ya hay instalada otra aplicación.
ANTIAFINIDAD CONSIGO MISMA (preferida)

NODO 1
    replica-1-wp
NODO 2
    replica-2-wp
NODO 3
    replica-3-wp

Apache + Wordpress (3 réplicas)

Todas esas reglas se pueden configurar como "preferidas" o "obligatorias"
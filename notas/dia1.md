
# Procedimientos de instalación de software

## Método tradicional: Instalación a pelo... sobre el OS del Hierro

          App1 + App2 + App3                 Problemas:
    ---------------------------------           - Incompatibilidad en los programas o dependencias o configuración
                 SO                             - Si app1 tiene un bug (100% CPU) -----> App1 = OFFLINE
    ---------------------------------                                                    App2 + App3 = OFFLINE
               Hierro                           - Potenciales huecos de seguridad

## Máquinas virtuales

       App1    |   App2 + App3               Me resuelve los problemas de las instalaciones a hierro.
    ---------------------------------           
       SO 1    |      SO 2                   El problema es que no es gratis:          
    ---------------------------------           - Merma en el rendimiento
       MV 1    |      MV 2                      - Merma de recursos 
    ---------------------------------           - Complejidad de gestión
      Hipervisor: esXI, Citrix
      kvm, HyperV..  
    ---------------------------------           
                 SO                             
    ---------------------------------           
               Hierro                           

## Soluciones un poquito más elaboradas

- Solaris Zones
  Lo que me permiten es crear entornos aislados dentro de un mismo SO.
  Se aprovechaba virtualización a nivel de hardware (procesadores Sparc de Sun)

## Contenedores

Son un entorno aislado dentro de un SO Linux donde puedo ejecutar procesos.
Aislado?
- Los contenedores, como entornos que son, tiene sus propias variables de entorno
- Tiene su propio FileSystem, independiente del SO anfitrión
- Tienen su propia configuración de red (IP, puertos, etc)
- Pueden tener limitaciones de acceso a recursos (CPU, memoria, disco)

       App1    |   App2 + App3                Esto me resuelve los mismo problemas que las VMs con respecto a las instalaciones a hierro
    ---------------------------------         Pero sin el sobrecoste de las VMs => VMs no más!
       C 1     |      C 2                   
    ---------------------------------         Las VMs siguen y seguirán... pero ahora las uso solo en el 5% de los casos en los 
      Gestor de contenedores:                 que antes las usaba.
      Docker, Podman, (Crio, Containerd)
    ---------------------------------           Los contenedores son una alternativa a las VMs para desplegar
           SO (Kernel Linux)                    software en un entorno aislado
    ---------------------------------           
               Hierro                           

Cuando trabajamos con contenedores NO INSTALAMOS software... lo desplegamos!

Los contenedores los creamos desde imágenes de contenedor.
Esas imágenes las descargo desde un registro de repositorios de imágenes dee contenedor:
- Docker hub
- Microsoft artifact registry
- Oracle container registry
- Quay.io <- Redhat registry

# Imágenes de contenedor

Una imagen de contenedor es un triste archivo comprimido (.tar) que tiene dentro:
- Una estructura de carpetas según estandar POSIX (No es obligatorio, pero todo el mundo lo hace así)

    bin/
        ls
        mv
        cp
    etc/
        nginx/
              /nginx.conf
    lib/
    usr/
        bin/
            apt
    opt/
        nginx/
               nginx <<< Ejecutable
    var/
    ...
- En la imagen no vienen solamente los programas gordos.... también vienen muchos programas de utilidad: ls, sh, bash, cp
- Metadatos de la imagen:
  - Puertos que por defecto abre el programa principal
  - Carpetas que usa el programa principal para guardar datos (volúmenes)
  - Valores por defecto para variables de entorno
  - Comando de arranque del programa principal
Programas ya instalados de antemano y configurados en esas carpetas

## Sistema de archivos de un contenedor

Hemos descargado una imagen de contenedor. Esa imagen tiene dentro una estructura de carpetas POSIX.

HOST: /
        bin/
            ls
            mv
            cp
        etc/
        lib/
        usr/
        opt/
        var/
            lib/
                docker/
                    containers/
                        mi-nginx/   <--- Contenedor corriendo NGINX
                             var/
                                logs/
                                    nginx.log
                             etc/
                                nginx/
                                    /nginx.conf
                        mi-nginx2/  <--- Contenedor corriendo NGINX
                             var/
                                logs/
                                    nginx.log
                             etc/
                                nginx/
                                    /nginx.conf
                    images/
                        nginx:1.23.4/   <--- Descomprimido de la imagen del contenedor (Ver 1)
                                        bin/
                                            ls
                                            mv
                                            cp
                                        etc/
                                            nginx/
                                                /nginx.conf
                                        lib/
                                        usr/
                                            bin/
                                                apt
                                        opt/
                                            nginx/
                                                nginx <<< Ejecutable
                                        var/
                                        ...

(1) Lo único que hacemos es engañar a los procesos que corren dentro del contenedor, haciéndoles creer que la raíz del sistema de archivos es la carpeta donde se descomprimió la imagen del contenedor. Pero eso ees algo que llevamos haciendo 43 años!
- chroot

Esa carpeta es inmutable. Nadie puede tocar ni escribir, ni modificar nada dentro de ella.
Cuando creamos un contenedor, se genera "otra carpeta" a parte de esa
Cualquier cambio sobre el sistema de archivos que venía en la imagen del contenedor, se escribe en esa otra carpeta.
Posteriormente lo que se presenta a los procesos que corren dentro del contenedor es la superposición de ambas carpetas.

La gracia de esto es que si creo 10 contenedores desde la misma imagen, los 10 contenedores comparten la misma carpeta inmutable (la de la imagen del contenedor). o necesito 2 carpetas del programa y utilidades.

Al descargar la imagen (.tar) se descomprime en una carpeta del SO anfitrión (Linux).
## Que puedo poner a correr en cun contenedor?

Lo que quiera... siempre que sea software que pueda correr en Linux (kernel Linux).
Todo software de carácter empresarial hoy en día se distribuye en forma de imagen de contenedor.


## Quiero instalar mongo en MONGO en Windows... tradicional

1. Hacerme con un INSTALADOR de mongo para Windows
2. Preparar el entorno (dependencias, configuraciones previas)
3. Ejecutar el instalador (puede ser más o menos compleja) -> Genera una instalación de mongo en el SO Windows
       c:\Archivos de programa\MongoDB\...
                                       ^^^^ Y la meto en un zip y os lo mando por email.
 


Se identifican por una URL, en la forma: REGISTRY /repo         :TAG
                                         docker.io/library/nginx:1.23.4
Los gestores de contenedores llevan sus propios registros preconfigurados. Si al descargar una imagen no pongo en su URL el REGISTRY, el gestor de contenedores buscará la imagen en su registro preconfigurado.


## TAGs de imágenes de contenedor

Es lo que identifica realmente a la imagen dentro del repositorio.
Que se usa como tag? Cada repositorio (empresa que lo gestiona) usa lo que que quiere, pero hay ciertos elementos que habitualmente encontramos en los tags:
- Versión del software principal que corre dentro del contenedor: nginx            TAG: 1.23.4
- Versiones de otros productos que existen dentro como dependencias: tomcat, la versión de java,  TAG: 9.0.112-jdk11
- Hay otra cosa rara que solemos encontrar aquí... y que a veces nos despista un huevo: Distros de linux:
  - bionic/focal (Ubuntu)
  - alpine
  - fedora
  - debian
  Eso solo me informa de las utilidades que voy a encontrar dentro de la imagen del contenedor. 
   Si tendré bash o no... Si tendré yum o apt o nada!
Por defecto, si no pongo tag, se usa el tag "latest"
OJO: No es una palabra mágica... es otro tag sin más, que muchos fabricantes de imágenes de contenedor usan para identificar la última versión estable de su software. Pero cuidado:
1. Está considerado una muy mala práctica usar el tag latest en entornos de producción. Nunca sabes qué versión te vas a descargar.
2. No todo fabricante generar imágenes con el tag latest


Hay 2 tipos de tags.. y hay que tener cuidado con ello:
- Tags fijos           nginx:       1.23.4
- Tags variables       nginx        1.23
                                    1
                                    latest 

En un entorno de producción que me interesaría?
    1.23.4 <<< Más conservadora... Siempre me instala esa versión (TAG FIJO)
    1.23   <<< Esta es ideal: Me trae la funcionalidad que necesito (MINOR), pero con la mayor cantidad posible de bugs arreglados 
               Hoy me puede apuntar a la 1.23.4, Y dentro de un mes a la 1.23.6 (mejor)
    1      <<< Nunca.... Esa no me va a traer un breaking change.. pero puede traer nueva funcionalidad que yo no necesito
               con nuevos bugs
               Hoy me puede apuntar a la 1.23.4, y dentro de un mes a la 1.24.0 (nueva funcionalidad)
    latest <<< Esta nunca... El día de mañana, sin yo saberlo, puede apuntar a la versión 2.0.0.... Y que mi sistema deje de funcionar

### Versiones de software

Al versionar software se suele usar lo que llamamos el semver... Esquema de versionado semántico.
X.Y.Z

                    Cuándo suben?
X = MAJOR           Cuando se introduce lo que llamamos un breaking change
                    Cambios incompatibles con versiones anteriores
Y = MINOR           Nueva funcionalidad o funcionalidad marcada como deprecated (obsoleta)
                    Opcionalmente pueden venir bugs arreglados
Z = PATCH           Cuando hay arreglos de bugs

Si paso de Oracle 12c a Oracle 19c... estoy ante un cambio mayor (major) incluye breaking changes
Eso implica que posiblemente necesite tocar (o no) mi programa porque hay funcionalidades de oracle que han sido eliminadas o reemplazadas por otras.

### Imagenes base de contenedor

Hay un tipo de imágenes que llamamos imágenes base: Ubuntu, Debian, Alpine, Fedora, CentOS, Oracle Linux

Esas imágenes no traen ningún sistema operativo dentro.
Qué traen y para qué sirven?

Traen:
La estructura de carpetas posix mínima: /bin, /etc, /lib, /usr, /var...
Algunos programas básicos: sh, ls, cp, mv..., de posix
Y algunos servicios típicos de esa distro linux: apt, dnf, bash

Alpine no trae nada... está pelada... los 20 comandos básicos de posix... y ya está.
En la de ubuntu trae los 20 comandos básicos de posix + apt + bash
En la de fedora trae los 20 comandos básicos de posix + dnf + bash

Eso es lo que viene ahí...

Se usan para crear imágenes de contenedor personalizadas con mis propias aplicaciones.

Lo normal en los entornos de producción es tirar de las imágenes alpine. Ocupan mucho menos.

Docker definió un estándar para crear imágenes de contenedor: Dockerfile

En esos archivos automatizamos la creación de imágenes de contenedor personalizadas.
Esto es lo que usa la gente de ElasticSearch para generar su imagen de contenedor oficial de ElasticSearch.

O lo que usaré yo para generar mi imagen de contenedor de mi microservicio para mi empresa.

```Dockerfile
FROM  fedora:37

RUN dnf install -y  java-11-openjdk-devel
COPY  mi-microservicio.jar   /opt/mimicroservicio/

CMD  ["java", "-jar", "/opt/mimicroservicio/mi-microservicio.jar"]
```

$ docker build -t mimicroservicio:1.0  .
$ docker image push mimicroservicio:1.0  myregistry.local/mimicroservicio:1.0
                                         aws me ofrece un registry privado para mis imágenes de contenedor
                                         artifactory
                                         nexus

---

# Redes en contenedores

En toda máquina encontramos interfaces varias:
- ens(eth): Interfaces de red físicas                   192.168.0.0/16 | 172.0.0.0/32
- lo: Interfaz de red de loopback (fqdn localhost)      127.0.0.0/8     <- IP: 127.0.0.1 (fqdn: localhost)
    Ésta es una red virtual, que solo existe dentro de mi host... y que permite comunicaciones entre programas que corren dentro de mi máquina sin necesidad de usar la red física.

Al instalar docker, por defecto Docker crea otra red virtual adicional (y posteriormente puedo montar otras 50 si quiero)
Es una red similar a la de loopback, pero que permite la comunicación entre contenedores y entre contenedores y el host.
La por defecto se monta en el rango  172.17.0.0/16
 Nuestro contenedores toman IPs secuenciales en ese rango, comenzando por 172.17.0.2
 La 1 la toma el host (y los contenedores lo usan de puerta de enlace)


    +----------------------------------------------------+------------------------------------ Red de amazon
    |                                                    |     
    172.31.10.12                                      MenchuPc - 172.31.10.20
    |    |
    |    +- NAT: Si se recibe una llamada en el puerto ???= 9999 en esta IP, quiero que se redirija al puerto 80 de la IP del nginx1
    |
    Mi máquina - 172.17.0.1 ---+------------------------------------- Red de docker (172.17.0.0/16)
    |                          |  
    127.0.0.1                  +-- 172.17.0.2 -- Contenedor 1: nginx abre puerto 80 de su IP -> 172.17.0.2:80
    |                          +-- 172.17.0.3 -- Contenedor 2: nginx abre puerto 80 de su IP -> 172.17.0.3:80
    |
    +---- Red de loopback

Menchu podría acceder a mi nginx1, mediante : 172.31.10.12:9999
A nivel de "Mi máquina", el puerto 9999 se redirige al puerto 80 de la IP del nginx1 = 172.17.0.2

Eso docker y otros gestores de contenedores lo hacen automáticamente por mí.... siempre que yo lo pida al crear el contenedor.

    $ docker container create --name nginx1 -p 172.31.10.12:9999:80 nginx:1.23.4
    172.31.10.12:9999:80
        172.31.10.12 La ip donde quiero configurar el nat
        9999         El puerto donde quiero configurar el nat
        80           El puerto que abre el nginx dentro del contenedor

    La IP de NAT no es obligatoria. Si no pongo valor, por defecto se usa: 0.0.0.0, que hace el nat en todas las IPs del host, IP de loopback incluido.

    Esto (lo de las redirecciones de puertos) lo usamos con 2 propósitos:
    - Poder acceder a los servicios que corren dentro de los contenedores desde fuera del host
    - Para ponérmelo yo más fácil si uso docker en local.


# Volúmenes de datos en contenedores

No es nada especial.
Simplemente es un punto de montaje adicional que hacemos dentro del fs del contenedor.
Como si hiciera un 
$ mount -t nfs nfs://servidor/carpeta  /punto/de/montaje/en/el/fs/contenedor

Esos puntos de montaje, docker y otros gestores de contenedores los gestionan por mí.. en el arranque del contenedor.

Para qué sirve esto?
- Persistir los datos tras el borrado del contenedor
- Compartir datos entre contenedores

      Weblogic (n)
        access.log      >  Filebeat > ES + Kibana (Monitorización logs)
                           fluentd
        
        Weblogic en un contenedor...
        Su archivo de log, lo quiero tener persistido (en Weblogic)... NO... la persistencia la quiero , pero en ES
        Eso si.. Necesito que otro programa (Filebeat) lea ese log y lo mande a ES
        Y ese programa lo ejecutaré en su propio contenedor (Filebeat)
        Necesito compartir el log de Weblogic (en una carpeta) con Filebeat (que corre en otro contenedor)

- Inyectar configuraciones o datos iniciales en el contenedor
  El contenedor viene con su propio nginx.conf... pero yo quiero usar mi propia configuración.
    Lo tengo en una carpeta nfs:/miservidor/nginx/config/nginx.conf
     -> mount nfs:/miservidor/nginx/config/  en /etc/nginx/ del contenedor
    Acabo de ponerle al nginx mi propia configuración.

Nos cuentan que en los contenedores no hay persistencia... pero de hecho SI QUE LA HAY..
Hay exactamente la misma persistencia que en una máquina física o virtual.
- Si reinicio una máquina física los datos siguen ahí? SI
- Si reinicio una máquina virtual los datos siguen ahí? SI
- Si reinicio un contenedor los datos siguen ahí? SI

Entonces doné viene el follón... El follón viene si borro el contenedor.. pero no es nada sorprendente:
- Si borro una máquina física los datos siguen ahí? NO
- Si borro una máquina virtual los datos siguen ahí? NO
- Si borro un contenedor los datos siguen ahí? NO

Cuál es la situación.. que al trabajar con contenedores, borramos contenedores de continuo...
Para una app, un contenedor se puede crear y borrar fácilmente 20 veces al día... en un entorno de producción!

Escenarios de ejemplo:
- Quiero actualizar un Mongo de la versión 4.4 a la 4.5
  Y lo tengo instalado a hierro.... Actualizar mongo de la v4.4 a la v4.5
  Y si lo tengo en una VM? Igual
  Y si lo tengo en un contenedor? YA NO.
    Lo que hago es borrar el contenedor que tengo de mongo v4.4
    Y creo un contenedor nuevo con la imagen de mongo v4.5
    Esto está guay.. pero .. y los datos?
        Aquí está el problema.
        TODO programa que va enlatado en contenedores, viene preparado para este tipo de upgrades automáticos.
        El problema son los datos.. que los pierdo al borrar el contenedor...
        Bueno.. o no.. depende donde estén guardados los datos.
        Si los guardo en el fs del contenedor, los pierdo.
        Si los guardo en un almacenamiento externo al contenedor, pero que he montado en el fs del contenedor (volumen de datos), los datos siguen ahí (siendo "ahí" donde leches sea fuera del fs del contenedor):
        - Carpeta nfs
        - Disco iSCSI
        - Almacenamiento en la nube (EBS, Azure Disk, Google Persistent Disk)
        - Una carpeta a nivel del host
- Escalabilidad horizontal. Estamos todo el día ajustando la infra a las necesidades de carga de trabajo.
  Iré creando contenedores y borrándolos según la carga de trabajo... que puede cambiar por minutos!
- Fallo en un servidor físico. Los contenedores que corrían en ese servidor se pierden. Necesito "mover" esos contenedores a otros servidores físicos. En realidad lo que hacemos es crear nuevos contenedores en otros servidores físicos. Y si algún día recuperamos el servidor físico que falló, borramos los que estaban ahí.

Herramientas tipo docker, me permiten fácilmente crear volúmenes de datos y montarlos en los contenedores.
Pero me permiten trabajar con tipos de volúmenes muy básicos:
- Carpetas dentro del host .. que comparto con el contenedor
- Volúmenes en Clouds
- Uso de Memoria RAM como almacenamiento temporal que monto en el contenedor (tmpfs)
- ...
- 3 cosas.

# Configuración de apps en contenedores

Dentro de una imagen de contenedor viene un programa ya instalado de antemano por alguien... normalmente el fabricante.
Ese programa va a traer la configuración que a mi me interesa? Ni de broma.. mucha casualidad!

Cómo puedo dar mis configuraciones a esos programas. 2 vías:
- Inyectar archivos de configuración en el contenedor (volúmenes de datos)
- Variables de entorno. Todo programa que va diseñado para correr dentro de un contenedor, permite que parte de su configuración se haga mediante variables de entorno.
  Ejemplo: Mongo
    MONGO_INITDB_ROOT_USERNAME=admin
    MONGO_INITDB_ROOT_PASSWORD=secret
---

# Entornos de producción

Qué diferencia un entorno de producción de otro tipo de entorno (de desarrollo, pruebas, etc)?
- Alta disponibilidad (REDUNDANCIA)
  Es tratar de conseguir que el sistema esté en funcionamiento al menos un determinado tiempo pactado previamente (SLA - Service Level Agreement).. Normalmente lo medimos en 9s... Es la carta a los reyes magos... Yo pido.. y veremos que llega!

        Disponibilidad del 90% = 36,5 días al año OFFLINE           | €
        Disponibilidad del 99% = 3,65 días al año OFFLINE           | €€
        Disponibilidad del 99.9% = 8,76 horas al año OFFLINE        | €€€€€
        Disponibilidad del 99.99% = 52,56 minutos al año OFFLINE    | €€€€€€€€€€€€€€€€€€€€
                                                                    |
                                                                    V

    Los 9s nos los tomamos más como una medida de la criticidad del sistema.

- Escalabilidad
  Capacidad de ajustar los recursos físicos a las necesidades del sistema en cada momento (carga de trabajo) 

    app1:  App departamental.
        día 1:         100 usuarios
        día 100:       100 usuarios    En un caso como este no necesito ESCALABILIDAD
        día 1000:      100 usuarios

    app2: 
        día 1:         100 usuarios
        día 100:     1.000 usuarios     En un caso como este necesito ESCALABILIDAD VERTICAL: MAS MAQUINA
        día 1000:   10.000 usuarios

    app3: Esto es internet
        día n:          100 usuarios
        día n+1:  1.000.000 usuarios     En un caso como este necesito ESCALABILIDAD HORIZONTAL: MAS MAQUINAS
        día n+2:         10 usuarios
        día n+3: 10.000.000 usuarios

    Soy la app de pedidos del telepi.
        00:00h usuario?         0 estoy cerrado
        09:00h? sigo cerrado 0
        11:00h? cuatros despistaos       10
        14:00h? hora punta comida        1.000
        17:00h? 3 cuampleaños            50
        21:00h... Madrid vs Barça de por medio.. Flipas! 1.000.000
        23:55h: Usuario 0

    Quién encaja con esto? Los clouds! Aportan infra cuando la necesito... con un modelo de pago por uso.
    Al final, comparto los servicios con otros clientes del cloud... 


- Monitorización
- Seguridad más avanzado


--- 

# Qué era UNIX?

Un SO que hacía la Americana de telecomunicaciones AT&T en los años 70 (lab. Bell). Se dejó de fabricar a principios de los 2000.
Se licenciaba de forma diferente a cómo se licencian los SO hoy en día (EULA-End User License Agreement).

AT&T licenciaba UNIX A grandes fabricantes de Hardware y a universidades.
Esos fabricantes adaptaban ese SO a su hardware y lo vendían con sus máquinas.

Follón.. llegó a haber más de 400 versiones diferentes de UNIX. Y empezaron a presentar incompatibilidades entre ellas.
Pusieron orden en ello, mediante 2 estándares:
- POSIX: Instituto de ingenieros eléctricos y electrónicos (IEEE)
- SUS (Single Unix Specification): The Open Group (TOG)

# Qué es UNIX?

Hoy en día nos referimos con UNIX a cualquier SO que cumple con los estándares POSIX y SUS.

Y muchos fabricantes de HW crean sus propias versiones de UNIX:

- HP: HP-UX (Unix®)
- IBM: AIX (Unix®) 
- Oracle: Solaris (Unix®)
- Apple: macOS (Unix®)

# Otros SO

Hay gente que en su momento trató de crear SO siguiendo esos estándares, pero gratuitos (sin certificarlos, que costaba pasta).

- Universidad de Berkeley: 386-BSD (Berkeley Software Distribution)
  La cagarón. Al decir que tenían un SO Unix... demanda de la gente de AT&T. Años de juicios. Al final le dieron la razón a BSD.. pero por entonces ya no usábamos micros 80386. Ese código se re-aprovechó después para otros SO:
    - FreeBSD
    - OpenBSD
    - NetBSD
    - MacOS (Apple se certifica como Unix® en 2007)
- Otros colegas... locos piraos (Richard Stallman)... al veer lo que pasó con la universidad de Berkeley, decidieron crear un SO , pero sin cagarla como ellos.
  - GNU: GNU's Not Unix
    Montaron todo lo que hacía falta para un SO:
    - Compiladores: gcc, make...
    - Editores de texto: emacs, vim...
    - Shells: bash, zsh...
    - Entornos gráficos: X11, GNOME
    - Juegos: chess
    - No valieron para montar una cosa: El kernel
    Proyecto parado.
- Linus Torvalds (1991): Crea un kernel supuestamente compatible con POSIX y SUS: Linux

Y paso lo que tenía que pasar... surge el amor:  GNU (70%) + Linux (30%) = GNU/Linux

Ese sistema operativo GNU/Linux es el que se ofrece en forma de distros: 
 RHEL, Fedora, Oracle Linux
 Debian -> Ubuntu, Mint
 Suse -> OpenSuse, SLES

GNU/Linux es un SO que creemos que en sus orígenes era compatible con POSIX y SUS... nunca se certificó como Unix®.
Y hoy en día sigue una linea de desarrollo totalmente diferente a la de los SO Unix® comerciales y sus estándares POSIX y SUS.

# POSIX

Es uno de esos estándares que definen qué es un SO Unix®.
Dentro de él se definen muchas cosas:
- Estructura típica de un Sistema de ficheros POSIX:
   /
     bin/
     etc/
     home/
     lib/
     usr/
     var/  
     tmp/
- Señales (kill )
- Permisos de ficheros (rwx + usuarios y grupos)
---

# Docker

$ docker <TIPO DE OBJETO> <VERBO>
            container       list rm create          start stop
            image           list rm pull 
            volume          list rm create
            network         list rm create

---

# SO Linux

Cualquier entorno que me permita correr un kernel Linux

GNU/Linux = SO
- Distribuciones: Ubuntu, CentOS, Debian, Fedora, RHEL, Suse...

Windows permite correr un kernel linux? Nativamente si, WSL2 <= Esto Microsoft lo montó para precisamente permitir la ejecución de contenedores Linux en Windows.

La apuesta dee Microsoft por Linux y contenedores hoy en día es total (fueron, junto con Redhat y AWS, los primeros en firmar acuerdo con Docker Inc. cuando sale como startup independiente en 2013).

---

Docker, Podman, Containerd, CRIO son gestores de contenedores que permiten crear, gestionar y borrar contenedores en un SO Linux.
Esto no vale para un entorno de producción. Ni nos ofrece HA, ni escalabilidad...

En entornos de producción necesitamos otra cosa: Kubernetes

# Kubernetes: 

~~Orquestador de contenedores~~

Es una herramienta que nos ofrece un lenguaje para DEFINIR entornos de producción basados en contenedores.
Y una vez definido es entorno, Kubernetes, lo crea y lo mantiene en funcionamiento 24x7.

Nos elimina a los administradores de sistemas de la empresa. YA NO HACEN FALTA. 
Me quedo con 2... que administren el cluster.


---

# Entornos de producción
            
                                                                            DNS

    Cluster                     Cluster de
    MariaDB                     Wordpress
    Galera
        MariaDB1  <<           Apache Httpd - 1  <<<
                                    Wordpress     
        MariaDB2  <<   BC   << Apache Httpd - 2  <<<  Balanceador de carga  <<<  Proxy reverso  <<< Proxy  <<<< Cliente
                                    Wordpress             nginx                   nginx
        MariaDB3  <<           Apache Httpd - 3  <<<      haproxy                 haproxy
                                    Wordpress             httpd                   httpd

    Firewall a nivel de red... que me permita controlar la comunicación entre los distintos componentes del sistema.
    Volumen compartido entre los Wordpress.
        Si se sube un fichero a uno de los wordpress, que esté disponible para los otros.

    Volumen MariaDB... uno por cada instancia de MariaDB (no compartido)

    Para qué monto un cluster Activo-Activo de MariaDB? HA... me temo que no.
    - HA .. para eso monto un master/replica... más barato y sencillo
    - Escalabilidad (mejora rendimiento)
      Cual es el factor más limitante en las BBDD (cuello de botella)? Escritura a HDD (lo más lento)

      Si tengo las 3 BBDD contra el mismo volumen...mejora rendimiento een escrituras a disco? NO
      El problema es:
        Los WP van a escribir simultaneamente el mismo archivo? NO
        Cada archivo que se sube a un WP, WP lo guarda en HDD con un ID/nombre único.
        Hay problema dee rendimiento en leer 3 tios el mismo archivo? NO
        El problema está en los bloqueos a nivel de escritura sobre el mismo archivo.
        Las tablas de una BBDD se guardan en un archivo o uno por tabla? TABLESPACE = Varios archivos... Pero tengo 1 por tabla? NO RARO

        Los SO dan 2 formas de trabajar en archivos:
        - Acceso secuencial: Leer/escribir desde el principio hasta el final.. o si acaso añadir al final <<< WP
        - Acceso aleatorio: Leer/escribir en cualquier posición del archivo <<< BBDD

    Las BBDD en cluster no usan normalmente un volumen compartido (como si hacen los WP).
    Cada BBDD tiene su propio volumen... y no todas guardan los mismos datos.

    Mariadb1        Dato1 Dato2
    Mariadb2        Dato1 Dato3
    Mariadb3        Dato2 Dato3

    Al haber x3 en la infra, tengo una mejor máxima teórica en rendimiento de un 50% (no 300%). Es el peaje de la HA

    Con una instancia en 2 instantes de tiempo puedo guardar 2 datos distintos.
    Con 3 instancias en 2 instantes de tiempo puedo guardar 3 datos distintos


# Kubernetes

    Cluster                     Cluster de
    MariaDB                     Wordpress
    Galera
        MariaDB1  <<           Apache Httpd - 1  <<<
                                    Wordpress     
        MariaDB2  <<   BC   << Apache Httpd - 2  <<<  Balanceador de carga  <<<  Proxy reverso  <<< Proxy  <<<< Cliente
                                    Wordpress             nginx                   nginx
        MariaDB3  <<           Apache Httpd - 3  <<<         haproxy                 haproxy
                                    Wordpress             httpd                   httpd

Kubernetes está pensando para definir un entorno como este... donde las apps funcionen dentro de contenedores.
Pero en kubernetes hablamos de :
Clusters de apps:
    Cluster de wordpress: Deployment (todos comparten un mismo volumen)
    Cluster de MariaDB: StatefulSet  (Cada instancia tiene su propio volumen)
PersistentVolumes:
    PV1:     Volumen compartido entre los Wordpress.
    PV2, PV3, PV4: Volumen MariaDB... uno por cada instancia de MariaDB (no compartido)
NetworkPolicies:
    Reglas de firewall a nivel de red... que me permita controlar la comunicación entre los distintos componentes del sistema.
     - Que a los MariaDB solo pueda acceder el cluster de wordpress
     - Que a los Wordpress solo pueda acceder el balanceador de carga
     - ...
Ese proxy reverso en Kubernetes se denomina Ingress Controller
 Y las reglas que en él definimos se llaman Ingresses
  Ingress que diga: http://miapp.miempresa.com/*  -> redirección ->   balanceador de carga del cluster de wordpress

Los Balanceadores de carga en Kubernetes se les llama Service

Es decir.. lo cobroncetes de google (que son los que montaron las primeras versiones de Kubernetes) nos cambiaron los nombres a los componentes estandar que ya veníamos usando desde hacía décadas en entornos de producción. Pero SON EXACTAMENTE LOS MISMOS CONCEPTOS.

En Kubernetes lo único que hacemos es CONFIGURAR OBJETOS dentro de mi entorno de producción:
- Deployments
- StatefulSets
- PersistentVolumes
- PersistentVolumeClaims
- ConfigMaps
- Secrets
- Services
- Ingresses
- NetworkPolicies
- ...
Y así hasta un total de unos 25-30 objetos que facilita Kubernetes.
Son las piezas FUNDAMENTALES de un entorno de producción basado.

Distribuciones de Kubernetes, como OpenShift, Rancher, EKS (AWS), AKS (Azure), GKE (Google Cloud), etc...lo que me ofrecen son Tipos de objetos adicionales (Operadores) . Esos tipos de objetos adicionales que podemos crear gracias a los operadores se denominan CRDs (Custom Resource Definitions).

Esos objetos de definen en Archivos YAML.
Y nuestro trabajo es SOLAMENTE definir esos archivos, cargarlos en Kubernetes e irnos a CaboLoco a tomar unos cocolocos.. mientras Kubernetes se encarga de crear y mantener en funcionamiento el entorno de producción que hemos definido.

Lo que queremos es AUTOMATIZAR la creación y operación de entornos de producción (basados en contenedores).
A nosotros nos da igual prácticamente que los programas corran o no en contenedores... 
La gracia es que en el mundo de los contenedores TODO ESTA TAN ESTANDARIZADO que podemos automatizar la creación y operación de esos entornos de producción de forma muy sencilla.

Conozco el comando con el que se arranca un elasticSearch? npi
Conozco el comando con el que se arranca un mongo? npi
Conozco el comando con el que se reinicia un mariadb? npi
Conozco el comando con el que se instala un wordpress nuevo en el cluster? npi

Si están en contenedores si:
- docker container restart/start/stop/create... me la pela lo que sea... Da igual lo que hay dentro...
- SE OPERA TODO IGUAL! ==>> AUTOMATIZAR FACILMENTE. Esto es en lo que se basa Kubernetes
- Gracias a que todo está TAN ESTANDARIZADO, kubernetes sabe como operarlo.


---

# Automatizar

Crear una máquina (o modificar el comportamiento de una mediante programas) para que esa máquina haga el trabajo que antes hacía un humano.

Puedo automatizar el lavado de la ropa -> Lavadora
Que incluso puedo personalizar con programas de lavado (programa de ropa delicada, algodón a 90º...)

En nuestro caso, la máquina la tenemos = Computador (servidor físico o virtual)
Lo que haremos será crear programas!

Para nosotros Automatizar = Programar

Quiero hacer un programa que haga lo que antes hacía un humano (administrador de sistemas)

Los programas los creeamos usando algún tipo de lenguaje informático.

Y en el mundo del desarrollo de software, hablamos de paradigmas de programación.

Hoy en día, un administrador de sistemas no administra sistemas. Ha cambiado de rol. La industria ha evolucionado.
Hoy en día un administrador de sistemas programa aplicaciones (scripts) que administren sistemas.

En nuestro caso, le vamos a pasar instrucciones (órdenes, statements) a Kubernetes para que kubernetes administre el entorno de producción por mí.

ESTO ES UNA DIFERENCIA ENORME con respecto a VMware VCenter/Nutanix.
Con estas herramientas, nosotros seguimos siendo quienen operamos los entornos de producción.
Yo entro en VMware VCenter/Nutanix y soy yo quien crea las máquinas virtuales, soy yo quien la reinicia, soy yo quien instala el software, soy yo quien configura las redes, los discos, etc...

ESTO NO PASA CON KUBERNETES. Ese trabajo lo hace KUBERNETES por mí.

Esto va un paso más allá de definir un entorno de producción. También tengo que explicarle a kubernetes COMO QUIERO QUE opere ese entorno de producción durante los próximos 5 años.

Cuando doy órdenes, le explico tareas a un programa/computador, lo puedo hacer usando lo que se llama un lenguaje imperativo (un paradigma de programación).

Al trabajar con computadoras, estamos muy acostumbrados al uso de un lenguaje imperativo.

Eso de los paradiogmas de programación no es sino un nombre un tanto hortera que los desarrolladores de software usan para decir /explicar las diferentes formas que tienen de usar un lenguaje.
Pero en los lenguajes humanos también pasa.

> Felipe, SI (IF) hay algo que no sea una silla debajo de la ventana (CONDICIONAL)
    > Felipe, lo quitas < Imperativo
> Felipe, Si (IF) no hay una silla debajo de la ventana (CONDICIONAL)
    >   Felipe if not silla (silla==False) then GOTO IKEA ! 
    > Felipe, pon una silla debajo de la ventana          < Imperativo

Estamos muy acostumbrados al uso del lenguaje imperativo. Y ES UNA MIERDA ENORME! NO NOS GUSTA .. Es un fastidio!
Porque me hace olvidar mi objetivo... para hacerme centarme en CÓMO llegar a ese objetivo... y ahñi es donde la cosa se complica... y se guarrea!

Esto es horrible! eso, es lo que llevamos haciendo DECADAS en los scripts de la bash, ps1, python...

Hay otros paradigmas. El que a día de hoy está triunfando en el mundo IT es el paradigma declarativo.

> Felipe, debajo de la ventana tiene que haber una silla. Es tu responsabilidad. DECLARATIVO => Idempotente

No le digo a Felipe lo que debe hacer... lee digo cómo quiero las cosas.. Como son las cosas.

DECLARO EL ESTADO EN EL QUE QUIERO LAS COSAS. Es lo que a mi realmente me importa..
El cómo conseguirlo en este caso lo delego en Felipe. A partir de ahora es su problema.. no el mío.
si ya hay sillas, si no hay sillas, si no hay ventana... ES TU PROBLEMA...
Al final de lo que sea que hagas, quiero que haya una silla debajo de la ventana. ESTO ES TODO LO QUE YO DIGO.

Kubernetes usa un lenguaje declarativo.
Docker compose usa un lenguaje declarativo.
Terraform usa un lenguaje declarativo.
Ansible usa un lenguaje declarativo.
Spring (Java) usa un lenguaje declarativo.

Todas las herramientas que a día de hoy están triunfando en el mundo IT usan lenguajes declarativos.

El lenguaje declarativo lleva consigo aparejado el concepto de IDEMPOTENCIA.

Idempotencia es una propiedad matemática: Una operación es idempotente si al aplicarla varias veces, el resultado es el mismo que si se aplica una sola vez. Operación (x1 = multiplicar por uno)

En el mundo de la informática, decimos que un programa es idempotente si al ejecutarlo varias veces, el resultado es el mismo que si se ejecuta una sola vez => Da igual el estado inicial del sistema, siempre se llega al mismo estado final tras ejecutar el programa.


Al hacer programación imperativa, soy yo eeel que debe buscar esa idempotencia.


Nuestro trabajo con Kubernetes es DEFINIR / DECLARAR el entorno de producción que queremos.
El trabajo de kubernetes es CREAR y OPERAR ese entorno de producción de acuerdo a nuestras declaraciones/especificaciones.

Kubernetes es quien me garantiza la idempotencia del entorno de producción.

Todas esas declaraciones son las que haremos en ficheros YAML... Y NO HAY ALTERNATIVA... Ni con una UI como la de Openshift, ni con línea de comandos... TODO es mediante ficheros YAML.

Esos ficheros además, como programas que son (declarativos .. no imperativos... pero programas al fin y al cabo) los versionaré y los trataré como a cualquier otro programa de software (GIT)...

En kubernetes, tenemos un cliente de linea de comandos (kubectl) que nos permite interactuar con kubernetes.
$ kubectl apply -f mi-entorno-produccion.yaml
$ kubectl get     ???? # Para generar listados de lo que hay en el entorno de producción en un momento dado

Openshift tiene su propio cliente de línea de comandos (oc)... que tiene exactamente la misma funcionalidad que kubectl y sintaxis idéntica.

$ oc apply -f mi-entorno-produccion.yaml
$ oc get     ???? # Para generar listados de lo que hay en el entorno de producción en un momento dado

Solo añade algunos verbos adicionales.... que ya hablaremos de ello.

Ojo.. Openshift es kubernetes... y por ende, con el comando kubectl también puedo gestionar Openshift.

---

Lo que vamos a estar haciendo los 3 días que quedan.

1º Aprender bien YAML (30 min)
2º Entender los objetos de kubernetes, y plantear un despliegue de una app en kubernetes MOGOLLON DE HORAS

    Node √
    Namespace √
    Pod √
    Job  √
    Deployment √
    StatefulSet √
    DaemonSet √
    CronJob √
    ConfigMap √
    Secret    √
    PersistentVolume √
    PersistentVolumeClaim √
    Service (ClusterIP, NodePort, LoadBalancer) √
    Ingress √
    HorizontalPodAutoscaler √

    NetworkPolicy
    ServiceAccount
    Role
    RoleBinding
    ClusterRole
    ClusterRoleBinding  
    ResourceQuota √
    LimitRange √

    Eso nos obligará a entender COMO FUNCIONA INTERNAMENTE UN CLUSTER DE KUBERNETES:
    - Arquitectura de un cluster de kubernetes
    - Comunicaciones entre los distintos componentes
    - Gestión de datos < Provisionadores dinámicos. 
    
3º Y una vez hecho esto: Jugaremos con un cluster de Kubernetes y con Openshift (que es kubernetes + extras)

    La app con la que jugaremos será Wordpress... es un ejemplo... da igual... Cualquier otra cosa sería exactamente igual.

---

# Openshift

Lo más parecido es VMWare/Nutanix... Nos ayuda a crear entornos de producción.
En VMWare/Nutanix trabajamos con Machines Virtuales (VMs) y en Openshift trabajamos con (Pods).
Otra diferencia es que Kubernetes se encarga de la creación y de la operación del entorno de producción.

Openshift es solo una distro de Kubernetes, pero con muchas herramientas adicionales que facilitan la vida a los desarrolladores y a los equipos de operaciones.


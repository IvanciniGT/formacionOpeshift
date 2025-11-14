kubectl <verbo> <tipo-objeto> ????                --all-namespaces       
                                                  --namespace, -n <nombre-namespace>

<verbos>:
    get                
    describe           
    delete

<tipo-objeto>:
    Pod                       pod pods
    Service                   service services svc
    Deployment
    ReplicaSet
    StatefulSet
    DaemonSet
    ConfigMap
    Secret
    Ingress
    PersistentVolume            pv pvs
    PersistentVolumeClaim       pvc pvcs
    Namespace                   ns
    Node
    Job
    CronJob
    HorizontalPodAutoscaler
    NetworkPolicy
    ServiceAccount
    Role
    RoleBinding
    ClusterRole
    ClusterRoleBinding
    ResourceQuota
    LimitRange


# Securización SSL en Route de OpenShift

Tenemos 3 opciones:
- PassThrough (el wordpress ya habla por https con un certificado propio y emitido por una CA de confianza pública).
- Reencrypt (Si el Wordpress ya habla por https, pero queremos que el Proxy Reverso también lo haga con su propio certificado). Especialmente útil si tenemos un certificado wildcard o un certificado válido de una CA reconocida.
- Edge      (El wordpress no habla por https, sino por http, pero el Proxy Reverso sí lo hace con su propio certificado.)


                     https (1)                    https (2)
                    <------------------>           <----->
    POD Wordpress  < Servicio Wordpress < Proxy Reverso < Clientes
                          ClusterIP         Ruta
                                            (en Kubernetes se configura con el Ingress)
          8080            http                 443            https


---

# Service Account

Es una cuenta de servicio.
Contra el api de kubernetes (apiServer) quien habla es siempre un PROGRAMA (kubectl, oc, dashboard, operador, etc). Para ello ese programa debe autenticarse, presentando unas credenciales de una cuenta de servicio.

Cuando yo me conecto via kubectl, los datos de la cuenta de servicio que uso (de hecho yo no la uso... se la paso al kubectl, para que el kubectl se autentique con el apiServer) están en el fichero config.

Cuando me conecto al dashboard, necesito también pasar el token de una cuenta de servicio.

Hay muchos programas que pueden/deben/necesitan hablar también con el apiserver:
- Provisionadores de volúmenes persistentes (para preguntar por las pvcs que se van lanzando y para crear los pv)
- Gitlab CI/CD. Debe poder crear pods dentro del cluster dinámicamente.,.. y luego borrarlos.

Lo que les configuramos son roles.

## Role en kubernetes:

Listado de los verbos que una cuenta de servicio tiene permitido aplicar sobre cada tipo de objeto.

En kubernetes los pods no necesitan tener un serviceAccount asociado. Si no van a hablar con el apiServer, no es necesario. Solo es necesario si el pod va a hablar con el apiServer (por ejemplo, un provisionador dinámico de volúmenes persistentes).

En cambio en Openshift por decretazo, todo pod que se lance en Openshift tiene asociada una serviceAccount por defecto (la default del namespace donde se lanza el pod). Si no queremos que use esa, debemos crear otra y asignársela al pod.


---

# Politicas de red: NetworkPolicy

Mee permiten configurar quién puede llamar a los puertos de un pod.<- Ingress
A donde puede conectarse ese pod. <- Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name:       permitir-llamadas-desde-frontend
spec:
  podSelector:
    matchLabels:
      app:    backend-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app:    frontend-app
      ports:
        - protocol:   TCP
          port:       8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app:    database-app
      ports:
        - protocol:   TCP
          port:       3306
```

Os decía que esto es inmantenible en un cluster grande.
En un entorno grande, donde tengo 50 despliegues... cada uno con 5/10 pods, la cantidad de políticas que tengo que crear es enorme. Y al final además, acabo con que tal programa no mee funciona... y tengo que revisar todas las políticas de red para ver cual es la que me está bloqueando.

Qué se hace en la realidad?
  Istio / Linkerd

Estas herramientas son casi de uso obligatorio en entornos productivos grandes, sobre todo cuando trabajamos con microservicios.

Esas herramientas lo que hacen es autohackearme. Me hago a mi mismo un ataque de tipo Man in the Middle (MitM).

Lo que hacen ees inyectar en cada pod un contenedor extra (sidecar) que es un proxy reverso (normalmente basado en Envoy).

                                          https (TLS mútua)
    POD BBDD  <------> POD Tomcat (MS1) <------> POD Tomcat (MS2)  <-----> KAFKA
                        Certificado 1+            Certificado 2 +          Certificado 3 +
                        Clave Privada 1.          Clave Privada 2.         Clave Privada 3.

Y ahora multiplica eso por 200 microservicios... y tienes un lío tremendo de certificados y claves privadas.
Que tengo que configurar en los tomcats... Y que mantener... que regenerar cada 3 meses.... INMANTENIBLE

Eso con una herramienta como ISTIO / LINKERD se automatiza todo en 5 minutos.

    POD BBDD           POD Tomcat (MS1)          POD Tomcat (MS2)  <-----> KAFKA
    Cont Mariadb       Cont Tomcat               Cont Tomcat                Cont Kafka
       ^v                 ^v                         ^v                         ^v           (localhost)
    + Envoy <-------->  + Envoy   <---------->    + Envoy    <----------->   + Envoy  (Estas son las que se securizan)
    
    + Inyectar reglas de Iptables para forzar que TODO el tráfico de red que salga o entra en cada pod
      pase por el contenedor Envoy (sidecar)

...
Claro.. al instalar estos envoys... TENGO CONTROL TOTAL DE TODAS LAS COMUNICACIONES ENTRE PODS.
Y eso es una locura lo que puedo hacer a partir de ese momento:
- Puedo securizar TODAS las comunicaciones entre pods (TLS mútua) gratis... con certificados automáticos y con regeneración cada X días automática.
- Puedo aplicar una política similar a la de SELinux pero entre comunicaciones de PODs: Modo permisivo (solo registro) o modo enforcing (bloqueo total)
- Funcionalidad extra en despliegues.
    Tengo app1 en v1.1.4... y sale la nueva versión de la app v1.2. Ese momento de darle ENTER! no tiene precio.
    Con Istio puedo hacer despliegues canary, blue/green, A/B testing, etc... de forma automática y sencilla.
     Puedo dejar las 2 versiones montadas een paralelo en el cluster.. que solo el 1% del tráfico me vaya a la nueva versión.
     Si va bien, subo al 10%, luego al 25%, luego al 50%... y finalmente al 100%
- Tener un cuando de mando visual (Kiali) que me permite ver todas las comunicaciones entre pods, tiempos de respuesta, errores, etc.
  Al trabajar con microservicios, es habitual que yo llamo a un microservicio que a su vez llama a otros 3 microservicios... y cada uno de esos llama a otros 2 microservicios... y al final tengo una cadena de llamadas entre microservicios que es larguísima.
  Que como algo falle... localizar el fallo es un infierno.




---

En Linux... para cuestiones de seguridad tenemos un montón de cosas:
- Permisos que poder aplicar a usuarios y grupos (chmod, chown, acl, etc)
- Con esos permisos podemos controlar acceso a ficheros y directorios.

SELinux(Redhat) / AppArmor (Ubuntu/Debian)

2 formas de trabajar: 
- Modo permisivo
- Modo enforcing
Cómo trabajamos con estas herramientas:
- Configuro mi servidor/instalo todas las cosas que necesito
- Pongo el modo permisivo
- Dejo que funcione un tiempo... y se me van anotando en el journal todos los accesos que se han realizado... y que se han permitido precisamente porque estoy en modo permisivo.
- Posteriormente pido un informe de todos esos accesos permitidos.
- Y en ese informe marco: Este si, Este no, Este si, Este no... Normalmente todo si. En revisar esta lista tardo poco
- Y pido entonces que se aplique el modo enforcing, pero con esas reglas que he aprobado.


---

Todo lo que sean productos comerciales, en docker hub u otros registries encuentro imágenes oficiales.
Con mucha probabilidad encuentro también charts oficiales en helm charts.

Pero y que pasa para nuestros desarrollos internos?
1º No tengo imágenes oficiales. La tengo que generar.
2º Tengo que dejar esa imagen de contenedor que yo estoy generando en un registry de contenedores:
   - Puede ser docker hub (público)... con usuario y contraseña.. cuenta pro.
   - Puede ser un registry privado (artifactory, nexus, harbor, etc)
   - Puede ser el registry interno que tiene Openshift (si uso Openshift) <<< Ventaja de usar Openshift
   - Kubernetes a priori no tiene registry interno. Tengo que usar uno externo.
     Aunque hay proyectos open source para montar un registry interno en kubernetes (harbor, etc) 
3º No habrá un chart de helm oficial:
   - Lo tengo que crear yo.
   - Usar archivos de manifiesto yaml que tengo que crear yo


Al final, Kubernetes no lleva:
- Plugin de red virtual
- Ingress Controller
- Registry de contenedores
- Sistema de recopilación de métricas avanzado (Prometheus + Grafana)
- Gestor de DNS externo
- No tiene provisionador de volúmenes persistentes avanzado 

En cambio, Openshift SI que lleva todo eso integrado.
---

Redhat de todo producto genera 2 subproductos paralelos:

                                                                                    Ansible Automation Platform
- Comercial (de pago + soporte)                  RHEL      JBOSS     Openshift      Ansible Tower

  Proyecto upstream                               ^

- Community (gratuito, sin soporte oficial)     Fedora     WildFly   OKD            AWX


Kubernetes oficial:
- K8S - Kubernetes base completo
- K3S - Kubernetes ligero para IoT y edge computing
- Minikube - Kubernetes para desarrollo en local (mi portátil)

Openshift oficial:
- Openshift Container Platform (OCP) - Comercial, de pago, con soporte oficial de Redhat
- OKD - Community, gratuito, sin soporte oficial de Redhat
- Minishift - Openshift para desarrollo en local (mi portátil) Hasta versión 3.x
  Openshift Local - Openshift para desarrollo en local (mi portátil) Desde versión 4.x 
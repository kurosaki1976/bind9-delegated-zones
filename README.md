# Configuración de un servidor DNS Bind9 con delegación de zonas

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Introducción

El objetivo de este tutorial es mostrar cómo configurar un servidor `DNS Bind9` que funcione como provedor (delegar responsabilidad de subdominios a otros servidores) y servidor con zonas delegadas. Como tutorial en sí, se le guiará a través de todo el proceso de configuración, pero se requieren conocimientos iniciales de `DNS` y `Bind9`. En las [referencias](#referencias), encontrará enlaces a sitios de `Internet` que le pueden ayudar.

> **NOTA**: La implementación de esta configuración ayuda a los administradores de red cubanos a mitigar la vulnerabilidad `DDoS por Amplificación de DNS`, notificada por la OSRI, como consecuencia de la recursividad en servidores DNS.

#### Descripción de las zonas y de la transferencia de zonas

El sistema de nombres de dominio [DNS](https://es.wikipedia.org/wiki/Sistema_de_nombres_de_dominio) permite dividir un espacio de nombres `DNS` en zonas, que almacenan información de nombres de uno o varios dominios `DNS`. Para cada nombre de dominio `DNS` incluido en una zona, la zona pasa a ser el origen autorizado de la información acerca de ese dominio.

Debido al importante papel que desempeñan las zonas en `DNS`, se pretende que éstas estén disponibles desde varios servidores `DNS` en la red para proporcionar disponibilidad y tolerancia a errores al resolver consultas de nombres. En caso contrario, si sólo se utiliza un servidor y éste no responde, se pueden producir errores en las consultas de nombres de la zona. Para que otros servidores alojen una zona, son necesarias transferencias de zona que repliquen y sincronicen todas las copias de la zona utilizadas en cada servidor configurado para alojar la zona.

#### Descripción de la diferencia entre zonas y dominios

Una zona se inicia como una base de datos de almacenamiento para un único nombre de dominio `DNS`. Si se agregan otros dominios bajo el dominio utilizado para crear la zona, estos dominios pueden formar parte de la misma zona o pertenecer a otra zona. Después de agregar un subdominio, se puede: administrarlo e incluirlo como parte de los registros de la zona original, o bien delegarlo a otra zona creada para admitir el subdominio.

Por tanto, se pueden definir dos métodos o estrategias de delegación de subdominios:

1. __Delegación completa de subdominios__

    En este caso, se deben agregar al archivo de zona directa del dominio superior, los correspondientes registros `NS`, según el tipo; los registros `A` ó `AAAA` -conocidos como (`glue records`)-, y uno o más servidores de nombres para los subdominios; así como crear los archivos de zona por cada inversa gestionada.

    > **NOTA**: Tiene como ventaja que cualquier cambio, sólo requerirá una recarga de la zona principal o el subdominio, respectivamente.

2. __Creación de subdominios virtuales o pseudo dominios__

    En este caso, se definirá la configuración de los subdominios, así como la configuración de la zona principal; en un mismo archivo de zona de dominio.

    > **NOTA**: Significa que la definición del dominio principal y la de los subdominios se incluyen en un archivo de zona única; no requiere nuevos servidores de nombres, ni registros `NS` ni `glue records` de tipo `A / AAAA`. Lo negativo es que cualquier cambio en la zona principal o en los subdominios requerirá una recarga total de la primera.

## Escenario

El escenario ejemplo lo constituye una red corporativa, donde el nodo principal administra la zona `example.tld` con direccionamiento `IPv4 172.16.0.0/16`, y ha decidido delegar la responsabilidad del subdomino `foo.example.tld` conjuntamente con la subred `172.16.23.192/29`. Teniendo en cuenta lo anterior podemos definir los siguientes parámetros:

1. __Dominio de nivel superior__

    * Zona `DNS`: `example.tld`
    * Zona inversa: `16.172.in-addr.arpa`
    * `FQND` servidor `DNS` primario: `ns1.example.tld`
    * Dirección `IP` servidor `DNS` primario: `172.16.0.18`
    * `FQND` servidor `DNS` secundario: `ns2.example.tld`
    * Dirección `IP` servidor `DNS` secundario: `172.16.2.18`

2. __Subdominio delegado__

    * Subdominio `DNS`: `foo.example.tld`
    * `FQND` servidor `DNS`: `ns.foo.example.tld`
    * Subred: `192/29.23.16.172.in-addr.arpa`
    * Dirección `IP` servidor `DNS`: `172.16.23.194`

## Administración del servidor `Bind9 DNS`

En las distribuciones `Debian GNU/Linux`, los ficheros de configuración del paquete `bind9`, se encuentran en `/etc/bind`. Ellos son: `named.conf` (fichero de configuración principal), `named.conf.default-zones` (contiene las zonas predefinidas de reenvío (`forward`), inversa (`reverse`) y difusión (`broadcast`) para el `localhost`), `named.conf.options` (contiene todos los parámetros para la operación del servicio), y `named.conf.local` (contiene las opciones de configuración y las declaraciones de zonas del servidor `DNS` local).

> **NOTA**: Es importante leer el archivo `/usr/share/doc/bind9/README.Debian.gz` antes de realizar modificaciones en los ficheros mencionados en el párrafo anterior, para obtener información sobre la estructura de los archivos de configuración del servicio `BIND` en `Debian`. De igual forma es una buena práctica realizar copias de seguridad de esos ficheros en su estado por defecto.

### Instalación

```bash
apt install bind9 dnsutils
```

> **NOTA**: Para disponer de la documentación `off-line`, instalar además `bind9-doc`.

### Configuración

#### Servidor `Bind9 DNS` primario de nivel superior

1. Ficheros de configuración

* `/etc/bind/named.conf`

```bash
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.log"
include "/etc/bind/rndc.key";
```

* `/etc/bind/named.conf.options`

```bash
options {
    version none;
    directory "/var/cache/bind";
    dnssec-enable yes;
    dnssec-validation yes;
    listen-on { 172.16.0.18; 127.0.0.1; };
    auth-nxdomain yes;
    listen-on-v6 { none; };
    empty-zones-enable no;
    allow-query { any; };
    recursion yes;
    minimal-responses yes;
    max-cache-size 128m;
    rate-limit {
        responses-per-second 15;
        log-only no;
    };
    flush-zones-on-shutdown yes;
    prefetch 0;
    allow-transfer { none; };
};

controls {
    inet 127.0.0.1 port 953
        allow { localhost; 172.16.0.18; } keys { rndc-key; };
};
```

* `/etc/bind/named.conf.local`

```bash
view "proveedor" {
    match-clients { localhost; 172.16.0.0/16; };
    recursion yes;
    allow-recursion { localhost; 172.16.0.0/16; };
    allow-recursion-on { localhost; 172.16.0.1; };
    include "/etc/bind/named.conf.default-zones";
    zone "example.tld" {
        type master;
        file "/etc/bind/db.example.tld";
        allow-transfer { 172.16.2.18; };
        also-notify { 172.16.2.18; };
        notify yes;
    };
    zone "0.16.172.in-addr.arpa" {
        type master;
        file "/etc/bind/db.0.16.172.in-addr.arpa";
        allow-transfer { 172.16.2.18; };
        also-notify { 172.16.2.18; };
        notify yes;
    };
     zone "2.16.172.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.2.16.172.in-addr.arpa";
        allow-transfer { 172.16.2.18; };
        also-notify { 172.16.2.18; };
        notify yes;
    };
    zone "23.16.172.in-addr.arpa" {
        type master;
        file "/etc/bind/db.23.16.172.in-addr.arpa";
        allow-transfer { 172.16.2.18; };
        also-notify { 172.16.2.18; };
        notify yes;
    };
};
```

2. Ficheros de declaración de zonas

* `/etc/bind/db.example.tld`

```bash
;
; example.tld DNS Domain Zone File
;
$ORIGIN .
$TTL 604800
example.tld IN  SOA ns1.example.tld. postmaster.example.tld. (
            2019091201  ; Serial
            3600        ; refresh [1h]
            600         ; retry [10m]
            1209600     ; expire [14d]
            3600        ; negative cache ttl [1h]
            )
;
        NS  ns1.example.tld.
        NS  ns2.example.tld.
        A   172.16.0.18
        A   172.16.2.18
        MX  10   mx.example.tld.
        TXT "v=spf1 ip4:172.16.0.10 a:mx.example.tld ~all"
;
$ORIGIN example.tld.
;
ns1  IN  A   172.16.0.18
mx   IN  A   172.16.0.10
ns2  IN  A   172.16.2.18
;
; subdominio delegado "foo.example.tld"
$ORIGIN foo.example.tld.
@  IN  NS  ns.foo.example.tld.
       A   172.16.23.194
ns IN  A   172.16.23.194
```

* `/etc/bind/db.0.16.172.in-addr.arpa`

```bash
;
; 172.16.0.0/24 Reverse Zone File
;
$ORIGIN .
$TTL 604800
0.16.172.IN-ADDR.ARPA IN  SOA ns1.example.tld. postmaster.example.tld. (
            2019091001  ; serial
            3600        ; refresh
            600         ; retry
            1209600     ; expire
            3600        ; negative cache ttl
            )
;
        NS  ns1.example.tld.
        NS  ns2.example.tld.
;
$ORIGIN 0.16.172.in-addr.arpa.
18    IN  PTR ns1.example.tld.
10   IN  PTR mx.example.tld.
```

* `/etc/bind/db.2.16.172.in-addr.arpa`

```bash
;
; 172.16.2.0/24 Reverse Zone File
;
$ORIGIN .
$TTL 604800
2.16.172.IN-ADDR.ARPA IN  SOA ns1.example.tld. postmaster.example.tld. (
            2019091001  ; serial
            3600        ; refresh
            600         ; retry
            1209600     ; expire
            3600        ; negative cache ttl
            )
;
        NS  ns1.example.tld.
        NS  ns2.example.tld.
;
$ORIGIN 2.16.172.in-addr.arpa.
18    IN  PTR ns2.example.tld.
```

* `/etc/bind/db.23.16.172.in-addr.arpa`

```bash
;
; 172.16.23.0/24 Reverse Zone File
;
$ORIGIN .
$TTL 604800
23.16.172.IN-ADDR.ARPA IN  SOA ns1.example.tld. postmaster.example.tld. (
            2019091001  ; serial
            3600        ; refresh
            600         ; retry
            1209600     ; expire
            3600        ; negative cache ttl
            )
;
        NS  ns1.example.tld.
        NS  ns2.example.tld.
;
; zona inversa delegada "172.16.23.192/29"
; usando "/" sintaxis de CIDR y macro $GENERATE
$ORIGIN 23.16.172.IN-ADDR.ARPA.
192/29  IN      NS      ns.foo.example.tld.
$GENERATE 192-199 $ CNAME $.192/29.23.16.172.IN-ADDR.ARPA.
; o abreviando
$GENERATE 192-199 $ CNAME $.192/29
```

> **NOTA**: La sintaxis `192/29` es un modo artificial (pero legítimo) para la construcción de zonas inversas delegadas. Antes del [RFC 2181](https://tools.ietf.org/html/rfc2181) el símbolo `/` no era un caracter legal para ser usado en el sistema de nombres de dominio, en su lugar se usaba el símbolo `-`.

Entonces es válido también realizar la delegación de esta forma:

```bash
; zona inversa delegada "172.16.23.192/29"
; usando "-" sintaxis de rango y macro $GENERATE
$ORIGIN 23.16.172.IN-ADDR.ARPA.
192-199  IN      NS      ns.foo.example.tld.
$GENERATE 192-199 $ CNAME $.192-199.23.16.172.IN-ADDR.ARPA.
; o abreviando
$GENERATE 192-199 $ CNAME $.192-199
```

#### Servidor `Bind9 DNS` secundario de nivel superior

1. Ficheros de configuración

* `/etc/bind/named.conf`

> Igual que el servidor primario.

* `/etc/bind/named.conf.options`

> Cambiar `listen-on { 172.16.2.18; 127.0.0.1; };` y `allow { localhost; 172.16.2.18; } keys { rndc-key; };`.

* `/etc/bind/named.conf.local`

```bash
view "proveedor" {
    match-clients { localhost; 172.16.0.0/16; };
    recursion yes;
    allow-recursion { localhost; 172.16.0.0/16; };
    allow-recursion-on { localhost; 172.16.2.18; };
    include "/etc/bind/named.conf.default-zones";
    zone "example.tld" {
        type slave;
        file "/etc/bind/db.example.tld";
        masters { 172.16.0.18; };
    };
    zone "0.16.172.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.0.16.172.in-addr.arpa";
        masters { 172.16.0.18; };
    };
    zone "2.16.172.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.2.16.172.in-addr.arpa";
        masters { 172.16.0.18; };
    };
    zone "23.16.172.in-addr.arpa" {
        type slave;
        file "/etc/bind/db.23.16.172.in-addr.arpa";
        masters { 172.16.0.18; };
    };
};
```

2. Ficheros de declaración de zonas

> Se obtienen por transferencia desde el servidor primario. Para forzar la transferencia de todas las zonas, ejecutar `rndc reload`, y para una específica `rndc reload 23.16.172.in-addr.arpa`. Para el buen funcionamiento de las transferencias, tiene que existir comunicación a través del puerto `tcp/udp 53`, entre los servidores.

#### Servidor `Bind9 DNS` con subdominio delegado

1. Ficheros de configuración

* `/etc/bind/named.conf`

```bash
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.log"
include "/etc/bind/rndc.key";
```

* `/etc/bind/named.conf.options`

```bash
options {
    version none;
    directory "/var/cache/bind";
    dnssec-enable yes;
    dnssec-validation yes;
    listen-on { 172.16.23.194; 127.0.0.1; };
    auth-nxdomain no;
    listen-on-v6 { none; };
    forwarders { 172.16.0.18; 172.16.2.18; };
    forward first;
    empty-zones-enable no;
    allow-query { any; };
    recursion no;
    minimal-responses yes;
    max-cache-size 128m;
    rate-limit {
        responses-per-second 15;
        log-only no;
    };
    flush-zones-on-shutdown yes;
    prefetch 0;
};

controls {
    inet 127.0.0.1 port 953
        allow { localhost; 172.16.23.194; } keys { rndc-key; };
};
```

* `/etc/bind/named.conf.local`

```bash
view "cliente" {
    match-clients { localhost; 172.16.0.0/16; };
    recursion no;
    include "/etc/bind/named.conf.default-zones";
    zone "foo.example.tld" {
        type master;
        file "/etc/bind/db.foo.example.tld";
        allow-update { none; };
        notify no;
    };
    zone "192/29.23.16.172.in-addr.arpa" {
        type master;
        file "/etc/bind/db.23.16.172.in-addr.arpa";
        allow-update { none; };
        notify no;
    };
};
```

> **NOTA**: Si se usa `-` (sintaxis de rango), la definición de la zona inversa sería `zone "192-199.23.16.172.in-addr.arpa"`.

2. Ficheros de declaración de zonas

* `/etc/bind/db.foo.example.tld`

```bash
;
; foo.example.tld DNS Domain Zone File
;
$ORIGIN .
$TTL 604800
foo.example.tld IN  SOA ns.foo.example.tld. postmaster.foo.example.tld. (
            2019091201  ; Serial
            3600        ; refresh
            600         ; retry
            1209600     ; expire
            3600        ; negative cache ttl
            )
;
        NS  ns.foo.example.tld.
        A   172.16.23.195
        MX  10   mail.foo.example.tld.
        TXT "v=spf1 ip4:172.16.23.195 a:mail.foo.example.tld include:example.tld ~all"
;
$ORIGIN foo.example.tld.
;
router IN  A   172.16.23.193
ns     IN  A   172.16.23.194
mail   IN  A   172.16.23.195
www    IN  A   172.16.23.196
jb     IN  A   172.16.23.197
ftp    IN  A   172.16.23.198
```

* `/etc/bind/db.23.16.172.in-addr.arpa`

```bash
;
; 172.16.23.192/29 Reverse Zone File
;
$ORIGIN .
$TTL 604800
192/29.23.16.172.IN-ADDR.ARPA IN  SOA ns.foo.example.tld. postmaster.foo.example.tld. (
            2019091001  ; serial
            3600        ; refresh
            600         ; retry
            1209600     ; expire
            3600        ; negative cache ttl
            )
;
        NS  ns.foo.example.tld.
;
$ORIGIN 192/29.23.16.172.IN-ADDR.ARPA.
193   IN  PTR router.foo.example.tld.
194   IN  PTR ns.foo.example.tld.
195   IN  PTR mail.foo.example.tld.
196   IN  PTR www.foo.example.tld.
197   IN  PTR jb.foo.example.tld.
198   IN  PTR ftp.foo.example.tld.
```

> **NOTA**: Si se usa `-` (sintaxis de rango), la definición de la zona inversa sería `192-199.23.16.172.IN-ADDR.ARPA`.

#### Para ambos servidores `Bind9 DNS`

Fichero `/etc/bind/named.conf.log` para almacenamiento de trazas, bitácora de eventos.

```bash
logging {
    channel "named-query" {
        file "/var/log/named_query.log" versions 1 size 5m;
        severity debug 3;
        print-time yes;
    };
    channel "debug" {
        file "/var/log/named.log" versions 1 size 3m;
        print-time yes;
        print-category yes;
    };
    category "queries" { "named-query"; };
    category "client" { "debug"; };
    category "config" { "debug"; };
    category "database" { "debug"; };
    category "default" { "debug"; };
    category "dispatch" { "debug"; };
    category "dnssec" { "debug"; };
    category "general" { "debug"; };
    category "lame-servers" { "debug"; };
    category "network" { "debug"; };
    category "notify" { "debug"; };
    category "resolver" { "debug"; };
    category "security" { "debug"; };
    category "unmatched" { "debug"; };
    category "update" { "debug"; };
    category "xfer-in" { "debug"; };
    category "xfer-out" { "debug"; };
};
```

## Comprobaciones

1. Comprobar la existencia de errores tanto en la configuración como en los ficheros de zonas.

```bash
named-checkconf -z
```

* Servidor `DNS` nivel superior

```bash
named-checkzone example.tld /etc/bind/db.example.tld
named-checkzone 0.16.172.IN-ADDR.ARPA /etc/bind/db.0.16.172.in-addr.arpa
named-checkzone foo.example.tld /etc/bind/db.example.tld
named-checkzone 23.16.172.IN-ADDR.ARPA /etc/bind/db.23.16.172.in-addr.arpa
```

* Servidor `DNS` subzonas de dominio e inversa delegadas

```bash
named-checkzone foo.example.tld /etc/bind/db.foo.example.tld
named-checkzone 192/29.23.16.172.IN-ADDR.ARPA /etc/bind/db.23.16.172.in-addr.arpa
```

> **NOTA**: Si se usa `-` (sintaxis de rango), la comprobación sería `named-checkzone 192-199.23.16.172.IN-ADDR.ARPA /etc/bind/db.23.16.172.in-addr.arpa`.

2. Reiniciar el servicio y realizar comprobaciones.

```bash
systemctl restart bind9.service
tail -fn100 /var/log/syslog

dig example.tld
host ns2.example.tld
dig foo.example.tld
host ns.foo.example.tld
host 172.16.23.194
dig +short 194.192/29.23.16.172.in-addr.arpa ptr
```

## Conclusiones

- Cuando se utiliza las funcionalidad de vistas, todas las definiciones de zonas **TIENEN** que estar contenidas dentro de éstas, de lo contrario el servicio no iniciará y generará códigos de error. Es por ello que -__en ambas configuraciones__-, el fichero **`/etc/bind/named.conf.default-zones`** fue incluido en cada definición de vista y no en el fichero de configuración principal.

- En el caso de la delegación de zonas, la recursividad es obligatoria en el ámbito del proveedor, sin importar el uso o no de la funcionalidad de vistas. Ello se debe a que la consulta es redirigida desde el servidor `DNS` responsable de la zona principal hacia el responsable de la subred delegada, y viceversa.

- Las configuraciones mostradas en esta guía utilizan la estrategia de __delegación completa de subdominios__, y son aplicables a los entornos de red `VPN` corporativas existentes en Cuba.

- La delegación completa tiene como inconveniente que si los servidores de nombre del subdominio delegado llegan a estar inalcanzables, la resolución no tendrá éxito; por eso es común encontrarnos en Cuba con configuraciones que incluyen la transferecia de zonas.

## Referencias
* [Bind9 - Debian Wiki](https://wiki.debian.org/Bind9)
* [HOWTO - Delegate a Sub-domain (a.k.a. subzone)](http://www.zytrax.com/books/dns/ch9/delegate.html)
* [HOWTO - Configure Sub-domains (a.k.a subzones)](http://www.zytrax.com/books/dns/ch9/subdomain.html)
* [3. DNS Reverse Mapping](http://www.zytrax.com/books/dns/ch3/)
* [$GENERATE Directive](http://www.zytrax.com/books/dns/ch8/generate.html)
* [Internet Systems Consortium](https://www.youtube.com/user/ISCdotorg/videos)
* [Clarifications to the DNS Specification](https://tools.ietf.org/html/rfc2181)
* [Domain Name System Structure and Delegation](https://tools.ietf.org/html/rfc1591)
* [Classless IN-ADDR.ARPA delegation](https://tools.ietf.org/html/rfc2317)
* [Classless in-addr.arpa subnet delegation](https://kb.isc.org/docs/aa-01589)
* [Classless reverse address delegation](https://www.netdaily.org/tag/29-classless-reverse-delegation-bind/)
* [Configure BIND as a slave DNS server](http://www.microhowto.info/howto/configure_bind_as_a_slave_dns_server.html)

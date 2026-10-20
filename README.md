# Guía paso a paso: configurar un servidor NTP con Chrony en Ubuntu

Vamos a configurar tu arquitectura para que el servidor A y el servidor B sincronicen sus relojes utilizando un tercer servidor NTP, mediante el software `chrony`.

La idea es que puedas realizar las pruebas de latencia de OpenDDS y RustDDS utilizando una misma referencia de tiempo.

Asumiré que los tres computadores tienen Ubuntu, están conectados a una misma red local y que la sincronización debe funcionar incluso sin conexión a Internet.

## 1. Arquitectura que vamos a configurar

Usaremos estas IP como ejemplos; debes reemplazarlas por las direcciones reales de tus servidores.

| Equipo       | IP de ejemplo  | Función                               |
| ------------ | -------------- | ------------------------------------- |
| Servidor NTP | `192.168.1.10` | Proporcionar una referencia de tiempo |
| Servidor A   | `192.168.1.11` | Publisher DDS                         |
| Servidor B   | `192.168.1.12` | Subscriber DDS                        |

## 2. Comprobar la conectividad entre los servidores

Ejecutar en A y B

Primero, comprueba que ambos servidores pueden comunicarse con el servidor NTP.

En el servidor A:

```
ping -c 4 192.168.1.10
```

Repite el mismo comando en el servidor B.

¿Qué estamos haciendo? Verificamos que existe conectividad básica por la red. Sin ella, los servidores no podrán consultar el reloj del servidor NTP.

Si recibes respuestas, podemos continuar. Si no, revisa las direcciones IP, las interfaces de red y las reglas de comunicación.

Nota: algunos servidores bloquean las solicitudes de ping, por lo que una falta de respuesta no siempre implica que NTP esté inaccesible.

## 3. Instalar Chrony en los tres equipos

Ejecutar en NTP, A y B

Como anteriormente tus servidores utilizaban `systemd-timesyncd`, vamos a utilizar `chrony` como único servicio encargado de sincronizar la hora.

Ejecuta:

```
sudo apt update
sudo apt install chrony
```

¿Qué estamos haciendo? Instalamos dos componentes:

- `chronyd`: servicio que sincroniza el reloj y también puede proporcionar la hora a otros equipos.
- `chronyc`: herramienta que permite comprobar y administrar el servicio.

En Ubuntu, instalar `chrony` normalmente sustituye al proveedor anterior de sincronización, evitando que ambos servicios intenten controlar el reloj simultáneamente.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://ubuntu.com\&sz=32)

Ubuntu Server documentation

+1



Si alguno de los equipos no tiene Internet, la instalación debe hacerse mediante paquetes disponibles en un repositorio interno o descargados previamente. Una vez instalado, `chrony` no necesita Internet para funcionar en nuestra arquitectura.

Comprueba que el servicio esté activo:

```
systemctl status chrony
```

Deberías encontrar:

```
Active: active (running)
```

También puedes verificar el servicio anterior:

```
systemctl is-active systemd-timesyncd
```

Este último no debería estar activo simultáneamente. Si todavía lo está, desactívalo con:

```
sudo systemctl disable --now systemd-timesyncd
```

## 4. Configurar el servidor NTP como referencia principal

Solo servidor NTP · 192.168.1.10

Este es el paso más importante porque vamos a convertir el tercer computador en la fuente de tiempo de toda la red.

### 4.1. Abrir el archivo de configuración

Ejecuta:

```
sudo nano /etc/chrony/chrony.conf
```

Este archivo contiene las instrucciones con las que funciona `chrony`.

### 4.2. Desactivar fuentes externas

Como estamos suponiendo una red aislada, no queremos que `chrony` intente consultar Internet.

Busca las líneas que comiencen con `pool` o `server` y hagan referencia a servidores públicos, por ejemplo:

```
pool ntp.ubuntu.com iburst
```

Coméntalas colocando `#` al comienzo:

```
# pool ntp.ubuntu.com iburst
```

En versiones recientes de Ubuntu, las fuentes pueden estar definidas también en archivos dentro de:

```
/etc/chrony/sources.d/
```

Revisa esa carpeta y desactiva las fuentes públicas de tiempo que encuentres. Conserva las demás opciones generales del archivo de configuración.

### 4.3. Activar la referencia de tiempo local

Agrega estas líneas en `/etc/chrony/chrony.conf`:

```
# Utilizar el reloj local como referencia
local stratum 10

# Permitir consultas de los dos servidores DDS
allow 192.168.1.11
allow 192.168.1.12
```

¿Qué significa cada instrucción?

- `local stratum 10`: permite que el servidor NTP utilice su propio reloj como referencia aunque no tenga una fuente externa.
- `allow`: autoriza a los servidores A y B a consultar su reloj.

El valor `10` es un nivel de referencia NTP, no una cantidad de segundos ni una medida de precisión.

Este modo de referencia local está diseñado precisamente para redes donde los equipos deben sincronizarse entre sí, aunque no tengan acceso a una fuente externa de tiempo.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://chrony-project.org\&sz=32)

chrony.conf(5)



### 4.4. Guardar y reiniciar Chrony

Guarda el archivo en Nano con `Ctrl + O`, `Enter` y `Ctrl + X`.

Después ejecuta:

```
sudo systemctl restart chrony
```

Y verifica:

```
systemctl status chrony
```

### 4.5. Permitir el tráfico NTP

Si el servidor utiliza el firewall UFW, comprueba su estado:

```
sudo ufw status
```

Si está activo, permite que A y B consulten el servidor por el puerto UDP 123:

```
sudo ufw allow from 192.168.1.11 to any port 123 proto udp
sudo ufw allow from 192.168.1.12 to any port 123 proto udp
```

¿Qué estamos haciendo? Permitimos la comunicación NTP únicamente desde los dos servidores DDS.

No necesitas abrir ese puerto al público ni permitir tráfico desde Internet.

Con esto, el servidor NTP queda configurado para ofrecer su reloj a los clientes.

## 5. Configurar el servidor A (Publisher)

Solo servidor A · 192.168.1.11

Ahora le indicaremos al Publisher que debe utilizar el servidor `192.168.1.10` como su única fuente de tiempo.

### 5.1. Abrir la configuración de Chrony

```
sudo nano /etc/chrony/chrony.conf
```

### 5.2. Configurar la fuente de tiempo

Desactiva las fuentes públicas NTP que puedan existir en este archivo o en `/etc/chrony/sources.d/`.

Agrega la siguiente línea:

```
server 192.168.1.10 iburst
```

¿Qué estamos haciendo?

- `server 192.168.1.10`: indica que el reloj del Publisher se sincronizará con el servidor NTP interno.
- `iburst`: permite realizar varias consultas iniciales para acelerar la sincronización.

En este equipo no necesitas agregar `allow` ni `local stratum 10`, porque el Publisher funcionará únicamente como cliente.

### 5.3. Reiniciar Chrony

```
sudo systemctl restart chrony
```

Con esto, el servidor A comenzará a consultar al servidor NTP.

## 6. Configurar el servidor B (Subscriber)

Solo servidor B · 192.168.1.12

El procedimiento es exactamente el mismo que utilizamos en el Publisher.

Abre el archivo:

```
sudo nano /etc/chrony/chrony.conf
```

Desactiva las fuentes públicas, tanto en este archivo como en `/etc/chrony/sources.d/`, si las hay.

Agrega:

```
server 192.168.1.10 iburst
```

Reinicia:

```
sudo systemctl restart chrony
```

¿Qué logramos? Tanto el Publisher como el Subscriber tienen configurada la misma fuente de tiempo, independientemente de si ejecutan OpenDDS o RustDDS.

## 7. Verificar que los servidores A y B están sincronizados

Esta es la parte más importante para tu proyecto de benchmarking.

No basta con que `chrony` esté instalado: necesitamos comprobar que se está utilizando correctamente.

### 7.1. Comprobar la fuente de sincronización

Ejecutar en A y B

```
chronyc sources -v
```

Deberías obtener una salida similar en ambos servidores:

```
MS Name/IP address    Stratum Poll Reach LastRx Last sample
===============================================================================
^* 192.168.1.10            10    6   377     20   +125us +/- 500us
```

Los datos relevantes son:

| Campo             | Qué debes comprobar                                                        |
| ----------------- | -------------------------------------------------------------------------- |
| `Name/IP address` | Debe aparecer `192.168.1.10`                                               |
| `^*`              | Significa que ese servidor fue seleccionado como fuente de tiempo          |
| `Reach`           | Indica si se han recibido respuestas recientes. `377` es un buen resultado |
| `Last sample`     | Presenta la desviación estimada y su margen de incertidumbre               |

El indicador principal es `^* 192.168.1.10`. Si aparece en ambos servidores, significa que los dos han seleccionado la misma referencia NTP.

Si aparece `^?`, la fuente todavía no es utilizable o no se están recibiendo respuestas válidas.

### 7.2. Comprobar el estado y la precisión estimada

Ejecutar en A y B

```
chronyc tracking
```

Ejemplo ilustrativo:

```
Reference ID    : C0A8010A (192.168.1.10)
Stratum         : 11
System time     : 0.000025 seconds slow of NTP time
Last offset     : -0.000012 seconds
RMS offset      : 0.000030 seconds
Root delay      : 0.000300 seconds
Root dispersion : 0.000100 seconds
Leap status     : Normal
```

Revisa especialmente:

- `Reference ID`: debe corresponder al servidor NTP interno.
- `System time`: corrección estimada pendiente en el reloj local. Idealmente debe ser pequeña.
- `Last offset`: diferencia estimada en la última actualización.
- `RMS offset`: refleja la magnitud de las desviaciones recientes.
- `Root delay` y `Root dispersion`: ayudan a evaluar la incertidumbre respecto a la referencia.
- `Leap status: Normal`: indica que Chrony considera válida la sincronización.

En este ejemplo, el valor `Stratum: 11` es esperado, porque nuestro servidor de referencia local anuncia el nivel 10.

### 7.3. Comprobar desde el servidor NTP que ambos clientes están consultándolo

Solo servidor NTP

Ejecuta:

```
chronyc clients
```

Deberías encontrar las direcciones de los dos servidores:

```
Hostname          NTP   Drop
192.168.1.11       25      0
192.168.1.12       24      0
```

Este ejemplo está simplificado.

¿Qué estamos comprobando? Que el servidor NTP ha recibido consultas de ambos clientes.

Esto complementa las verificaciones anteriores, aunque por sí solo no confirma que los relojes ya estén sincronizados.

## 8. ¿Cómo determinar si la precisión es suficiente para DDS?

Supongamos que, después de configurar todo, tienes estos resultados:

| Indicador           | Servidor A   | Servidor B   |
| ------------------- | ------------ | ------------ |
| Fuente seleccionada | 192.168.1.10 | 192.168.1.10 |
| Leap status         | Normal       | Normal       |
| System time         | +20 µs       | -30 µs       |
| RMS offset          | 40 µs        | 50 µs        |

Estos valores hipotéticos serían una señal positiva de que ambos servidores tienen desviaciones pequeñas frente a la referencia NTP.

Sin embargo, no permiten afirmar que la diferencia real entre sus relojes sea exactamente 50 µs. Para estimar el error entre los equipos debes considerar también la incertidumbre de sincronización.

Para tu benchmarking, procura que esa incertidumbre sea mucho menor que la latencia DDS que deseas medir.

## 9. Lista final de verificación

Verificación de la configuración

0/8

Servidor NTP

Chrony instalado y activo

Servidor NTP

Referencia local configurada con local stratum 10

Servidor NTP

Permite consultas NTP de A y B

Servidor A

Utiliza 192.168.1.10 como única fuente

Servidor B

Utiliza 192.168.1.10 como única fuente

A y B

chronyc sources -v muestra ^\* junto a 192.168.1.10

A y B

chronyc tracking muestra Leap status: Normal

Servidor NTP

chronyc clients muestra consultas de A y B

## 10. Consideración importante sobre utilizar el reloj local

Como elegimos una arquitectura sin Internet ni una fuente GPS, el servidor NTP utiliza su propio reloj para establecer la referencia.

Esto significa que:

- Los servidores A y B pueden sincronizarse entre sí.
- No necesariamente tendrán la hora real exacta.
- Si el reloj del servidor NTP se desvía, los clientes seguirán esa referencia.
- Si reinicias o modificas manualmente la hora del servidor NTP, puedes afectar las mediciones.

Para tu proyecto DDS, lo más importante es la sincronización relativa entre Publisher y Subscriber, siempre que la precisión obtenida sea suficiente para medir las latencias.

Mi recomendación es realizar primero esta configuración, comprobar el estado con `chronyc sources -v` y `chronyc tracking`, y registrar las desviaciones e incertidumbres antes y después de cada prueba.

Si posteriormente necesitas que el servidor NTP obtenga la hora de Internet, la arquitectura de los clientes no cambia: solo tendrás que configurar una fuente NTP externa en el servidor central en lugar de depender exclusivamente de su reloj local.

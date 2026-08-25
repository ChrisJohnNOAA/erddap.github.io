# Bloqueo ERDDAP™ solicitudes basadas en el contenido de la solicitud, no basadas en la IP

Este contenido se basa en un [mensaje de Roy Mendelssohn al ERDDAP™ Grupo de usuarios](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## El problema

Si eres como nosotros, estás viendo muchos bots haciendo solicitudes de datos a tu ERDDAP™ , y las solicitudes a menudo parecen ser hechas por scripts mal codificados, ya sea por LLMs (Supongo) o por humanos. Una cosa sobre LLMs es que harán exactamente lo que usted les pide que hagan, por lo que si usted no les dice que tengan los códigos de devolución de la comprobación del script, y no les diga qué hacer en diferentes casos, entonces generalmente el código no hará ninguna de esas cosas. Y si le dices que siga intentándolo, lo hará. Lo cual es lo que estamos viendo al menos.

Pero los que están haciendo un número en nuestro ERDDAP™ son así:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Estos envían ERDDAP™ en tierra nunca-nunca, debido, a lo mejor que puedo determinar, que esta URL no sólo está solicitando el conjunto de datos completo, que es que no sé cuántos terabytes, pero quiere que se convierta a un archivo de parquet. Peor que las solicitudes vienen de mover IPs, por lo que bloquear IPs es peor que whack-a-mole, nunca los bloqueará a todos.

Así que la pregunta es cómo puede bloquear basado en el contenido de la solicitud? Antes de ir más lejos, si usted está experimentando fallos puede ser de diferentes causas entonces lo que estamos viendo - Pasé mucho tiempo pasando por registros y también me notifican cuando hay un uso alto de la memoria peligroso, y notó cuando un accidente siguió esa notificación, y también notó cuál era el denominador común en estos fallos, que puede no ser el caso para su servidor. Así que investiga esto antes de tomar cualquier acción, pero esto puede darle alguna idea de cómo bloquear estas solicitudes.

Para que esté claro, puedes pensar en un ERDDAP™ solicitud como:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

En nuestro caso, el denominador común de los fallos fueron solicitudes de rejilla o mesa con ciertos tipos de archivo pero sin limitaciones. Así que la pregunta es cómo bloquear las solicitudes sin restricciones? Excepto que no es tan simple, porque hay algún número de tipos de archivos que no necesitan una restricción y se comportarán bien, así que no queremos bloquearlos. Después de hablar con Chris, y sin duda he dejado algo, los tipos de archivos que están bien comportados sin una restricción son:

.croissant
.iso19115_2
.iso19139_2007
.iso19115_3_2016
 .nc CFHeader
 .nc CFMAHeader
.das
.dds
HTML
.graph
.subset
 .nc Header
.ayuda
.fgdc
.iso19115
 .nc OJsonHeader (esta será en la próxima nueva versión) .

## Solución

La solución "Encontré" funciona en Apache2, no sé nginx pero imagino que hay algo similar. Al principio intenté mod_security, pero eso se puso demasiado complicado y no funcionó muy bien. La solución es usar mod_rewrite. Ahora soy todo menos un experto en esto, así que mientras definía lo que estaba tratando de lograr, la solución, así como la explicación, se debe a Claude. Ai. ChatGPT le proporcionará básicamente la misma respuesta.

Paso 1. Asegúrese de que mod_rewrite está instalado y habilitado. Como hacer esto varía con OS, esto es algo que puedes preguntar a tu chatbot favorito.

Paso 2. En el archivo apropiado que configura ssl para apache2 (que de nuevo varía por OS) , por ejemplo podría ser algo como /etc/apache2/sites-enabled/ssl.conf, añadir lo siguiente bajo la definición apropiada de VirtualHost (nota si copia esto sólo hay 4 líneas, la tercera línea puede ser envuelta, desenvolverlo) 

RewriteEngine On
RewriteCond %&#123;QUERY_STRING&#125; ^$
RewriteCond %&#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) /[^/?]+\\. (? (? | iso19115_2 | iso19139_2007 | iso19115_3_2016 | ncCFHeader | ncCFMAHeader | das | dds | html | Gráfico | subset | ncHeader | ayuda | fgdc | iso19115 | ncoJsonHeader) $) [A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]

Paso 3. Compruebe que la configuración es válida: sudo apache2ctl configtest

Paso 4. Reinicie apache2: sistema sudoctl reiniciar apache2

Paso 5. Revise sus registros que nada está siendo bloqueado que no debe ser, y que las peticiones apropiadas sin una restricción devuelve un 429 sin golpear nunca su tomcat

## Explicación

¿Por qué funciona esto y qué hace esto - aquí está la explicación de Claude.ai:

Línea 1 - RewriteEngine On
Activa el procesamiento mod_rewrite para este alcance. Sin ella, las directivas RewriteCond/RewriteRule son simplemente ignoradas.

Línea 2 - Reescribir Cond %&#123;QUERY_STRING&#125; ^$
Una condición que debe ser verdadera antes de que se aplique la regla siguiente. %&#123;QUERY_STRING&#125; es todo después de la ? en la URL de solicitud. ^$ es un regex que significa "el comienzo de la cuerda inmediatamente seguido por el final de la cuerda" — es decir, una cadena vacía. Así que esta condición es verdadera sólo cuando no hay ninguna cadena de consulta en absoluto — ninguna expresión subsetting/constraint a petición.

Línea 3 - Reescribir Cond %&#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) /[^/?]+\\. (? (?) $) [A-Za-z0-9_]+$
Una segunda condición, comprobada contra %&#123;REQUEST_URI&#125; — el camino de solicitud literal como el cliente lo envió, siempre el camino completo independientemente de dónde en el config esta regla vive (deliberadamente elegido sobre dejar el patrón de RewriteRule hacer el emparejamiento, porque el patrón-matching dentro de un <Location> bloque puede comportarse ambiguamente — vea la nota abajo) . Derribando el regex:

^/erddap/ — debe comenzar con /erddap/
 (griddap |  tabledap ) / - seguido de uno de los dos ERDDAP™ métodos de acceso
[^/?]+ datasetID : uno o más caracteres que no son / o ?
\\. — un punto literal
 (? (? | iso19115_2 | ... | ncoJsonHeader) $) — una mirada negativa: "siempre que lo que sigue no sea uno de estos archivos exactos Escribe nombres hasta el final de la cadena." Estos son los tipos de archivo ERDDAP™ puede servir legítimamente sin limitaciones (metadatos, estructura, páginas de formulario, etc.) — la cabeza es lo que los excluye de ser bloqueados.
[A-Za-z0-9_]+$ — el archivo real Tipo de extensión (letras, dígitos, subrayar) , requerido para correr hasta el final de la cadena.
Así que esta condición es verdadera sólo cuando el camino es un griddap/ tabledap solicitud de un archivo Tipo que no está en la lista de seguridad sin restricciones.

Línea 4 — RewriteRule ^ - [R=429,L]
La regla misma. Debido a que ambas condiciones anteriores ya deben ser ciertas para que Apache incluso evalúe esta línea, el patrón aquí no necesita comprobar nada más — ^ sólo coincide con "el comienzo de la cuerda", que siempre es cierto. - significa "no reescribas la URL a nada diferente" (no vamos a redireccionar en ningún lado, sólo cortocircuito de la solicitud) . Las banderas:

R=429 — responder con una acción de clase HTTP que lleva código de estado 429 ("Demasiadas peticiones") en lugar de servir la solicitud.
L — "Última": deja de procesar cualquier otra regla de reescritura una vez que este incendio.
Junta: si la cadena de consulta está vacía, Y la solicitud es para un griddap/ tabledap archivo Tipo que no está en la lista sin restricciones seguras, devuelve inmediatamente 429 — sin contacto con el ERDDAP /Tomcat backend.

¿Por qué %&#123;REQUEST_URI&#125; en lugar de dejar que el patrón de RewriteRule coincida directamente con el camino (vale la pena incluir como nota para los colegas, ya que es la parte no obvia) : dentro de un <Location> bloque, lo que un patrón de RewriteRule desnudo se combina en realidad puede comportarse incoherentemente dependiendo de la versión y el contexto de Apache. Explicadamente tirando del camino completo a través de RewriteCond %&#123;REQUEST_URI&#125; pasos que ambigüedad enteramente — siempre es el camino de solicitud literal y completo, por lo que el regex se comporta exactamente como escrito independientemente de dónde se anida la regla.


He estado usando esto durante varios días y parece funcionar muy bien, está bloqueando lo que estoy tratando de bloquear y no bloquear lo que no quiero bloquear. Y nuestra ERDDAP™ se ha vuelto mucho más estable.

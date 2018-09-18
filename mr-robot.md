---
layout: default
---

<br>
Bueno para comenzar bienvenidos a mi blog espero subir walkthroughs con soluciones a máquinas vulnerables,
obtenidas desde vulnhub.com, y luego subir máquinas propias.
Primero vamos a resolver una máquina virtual cuyo nombre es tentador para los fanáticos de esta serie de televisión,
Mr.Robot, así es máquina cuya imagen está disponible en el siguiente <a href="https://www.vulnhub.com/entry/mr-robot-1,151/">enlace</a>.

Acá podemos ver la máquina corriendo en VirtualBox:

<img class="image" src="/assets/img/mrRobot0.png" alt="">

Como primer detalle tenemos en el sitio oficial, que la dificultad es "beginner". Espera hay un detalle interesante a
pesar de ser para principiantes, no se nos da la clave de acceso, por lo que no se podrá "mirar" desde el punto de vista
de la máquina, un detalle agradable desde cierta perspectiva....

Se nos dan las siguientes indicaciones:

> La máquina tiene **tres** llaves  ocultas en diferentes locaciones, el objetivo es obtener las tres.
> A medida que se encuentren las llaves se aumentará la dificultad.

Segmentaré los pasos para mayor claridad, asi que manos a la obra!.

## Primera parte: Nmap y un escaneo de puertos.

Bueno lo primero que haremos será realizar un escaneo simple de puertos en la máquina objetivo, como se puede ver en la
imágen abajo.

{% highlight bash %}
$ nmap -sT -Pn www.mr-robot.com
{% endhighlight %}

![Captura 1](/assets/img/mrRobot1.png){: .image}

A lo que podemos inferir que los vectores de ataque posibles, en este caso podrían ser los puertos
* tcp/80 HTTP
* tcp/443 HTTPS

Ahora veamos que servicios son los que mantienen el sitio web arriba:

{% highlight bash %}
$ nmap -A -Pn www.mr-robot.com
{% endhighlight %}

![Captura 2](/assets/img/mrRobot2.png){: .image}

Ok no nos ha entregado tanta información como se esperaría asi que vamos a pasar a la siguiente etapa,
y entraremos al sitio web "hosteado" por la máquina virtual.

## Segunda Parte: Ojeando el Sitio Web...
Ahora tenemos varios "caminos" a seguir, por lo que visitaremos el sitio web y a ver que tal.

![Captura 3](/assets/img/mrRobot3.png){: .image}

Vaya, vaya esto ha sido una sorpresa grata!, el sitio tiene buena pinta!.....Así que te invito a probar los distintos
"comandos", cosa que omitiré de publicar para no quitarle la "magia" al proceso. Veamos que hay detrás de este sitio...

{% highlight bash %}
$ nikto -h www.mr-robot.com
{% endhighlight %}

Lo que arroja el siguiente resultado:

![Captura 4](/assets/img/mrRobot4.png){: .image}

Bah el sitio está bajo Wordpress, esto....Ahora veamos que nos dice el archivo robots.txt....

> User-agent: *
>
> fsocity.dic
>
> key-1-of-3.txt

Ajá, ya tenemos nuestra primer key ?!, veamos cual es entonces....

> 073403c8a58a1f80d943455fb30724b9

Mmm el siguiente archivo es un diccionario, no sabemos si contiene usuarios, contraseñas, así que a descargarlo se ha dicho y por que no ser un fanático de la shell y usar wget.

{% highlight bash %}
$ wget www.mr-robot.com/fsocity.dic
{% endhighlight %}

![Captura 6](/assets/img/mrRobot6.png){: .image}

Wopa! 6.9MB, me pregunto cual será el contenido de este diccionario, veamos cuantas líneas tiene...

{% highlight bash %}
> wc -l fsocity.dic
858160 fsocity.dic
{% endhighlight %}

Si vemos el final del archivo podremos notar que no están ordenados, y que hay repetidos, así que veamos si podemos reducir un poco este número.

{% highlight bash %}
> sort fsocity.dic | uniq >> diccionario.dic
{% endhighlight %}

Bingo!! hemos logrado reducir el número a...

{% highlight bash %}
> wc -l diccionario.dic
11451 diccionario.dic
{% endhighlight %}

Ahora a buscar más información sobre el sitio web. Si regresamos a los hallazgos realizados por nikto, tenemos algunos puntos más por revisar, por ejemplo veamos el contenido del readme.

{% highlight text %}
> curl www.mr-robot.com/readme
I like where you head is at. However I'm not going to help you.
{% endhighlight %}

Puff! nada interesante ?!, Entonces veamos el admin...Un bucle infinito ? ouch eso no era lo que esperaba pero bueno...
Así que escribamos una url aleatoria a ver que mensaje de error arroja y si nos da alguna pista sobre el funcionamiento interno. Usaré la siguiente URL: "www.mr-robot.com/asdfg"

![Captura 7](/assets/img/mrRobot7.png){: .image}

Genial! veamos si es posible realizar búsquedas...Nop nos redirecciona a la página principal, así que veamos el enlace al
login desde la página anterior o bien escribiendo la dirección en el navegador.

http://www.mr-robot.com/wp-login.php

Ahora probaremos un usuario típico en la mayoría de los sitios basados en WordPress, el "admin" y la contraseña
"admin123", en serio hay gente que la usa....

![Captura 8](/assets/img/mrRobot8.png){: .image}

Ok esto es interesante, por defecto Wordpress nos da información sobre la inexistencia o existencia del nombre de usuario, demos las gracias por esto....Arrojando lo siguiente: **Error: Invalid username**. Así que el "admin" nos ha fallado jjajaja no en serio habrá que usar el diccionario antes descargado. Pero antes veamos mediante Burp como trabaja este Wordpress.

Usamos el proxy y a ver que nos arroja.....

![Captura 9](/assets/img/mrRobot9.png){: .image}

Ahora veamos la pestaña target y tenemos lo siguiente....Ya se verá el motivo de verificar la respuesta del servidor, si te acomoda esto lo podrías hacer mediante las herramientas de desarrollador de chrome, chromium, firefox, etc.

![Captura 10](/assets/img/mrRobot10.png){: .image}

Y acá tenemos nuestro mensaje de error cuando el usuario no es válido.

{% highlight html %}
<div id="login_error">	<strong>ERROR</strong>: Invalid username.
{% endhighlight %}

Ya con estos datos, podremos hacer uso del diccionario mediante Hydra, como sigue:

{% highlight bash %}
$ hydra 192.168.56.102 http-form-post "/wp-login.php:log=^USER^&pwd=^PASS^
:invalid" -L diccionario.dic -p admin123
{% endhighlight %}

Donde según definimos anteriormente la clave puede ser cualquiera, lo que queremos es determinar algún usuario válido mediante el diccionario, ahora si te preguntas por que no simplemente usar wpscan y listar los usuarios, bueno es otro método y ya verás los resultados, también podrías querer definir en Hydra que usuario y contraseña usarán el mismo diccionario, pero tomará demasiado tiempo, por lo que te aconsejo este método.

![Captura 5](/assets/img/mrRobot5.png){: .image}

Que bien!! Tenemos usuarios?. Espera un momento, Elliot? era muy obvio?. Ahora como podemos ver en la imágen Wordpress no es sensible a mayúscula/minúscula...Y gracias a eso podemos elegir el usuario jajaja, usaré "elliot" todo en minúscula tu elige el que quieras. Y ahora lo bueno !! a usar Hydra para obtener la clave. Pero antes veamos que mensaje arroja la contraseña incorrecta con el usuario "elliot".

![Captura 13](/assets/img/mrRobot13.png){: .image}

Ajá veamos en Burp el código HTML asociado al error, en búsqueda de la palabra clave...

![Captura 12](/assets/img/mrRobot12.png){: .image}

Por tanto ahora sabemos que buscamos contraseñas para el usuario "elliot" que no arrojen **incorrect** en el cuerpo HTML de la respuesta. Usando Hydra debemos poner lo siguiente:

{% highlight bash %}
$ hydra 192.168.56.102 http-form-post "/wp-login.php:log=^USER^&pwd=^PASS^
:incorrect" -l elliot -P sorted.dic
{% endhighlight %}

Lo que nos retornará lo siguiente:

![Captura 11](/assets/img/mrRobot11.png){: .image}

Bingo!!, tenemos la contraseña y el usuario, entremos entonces!..

![Captura 14](/assets/img/mrRobot14.png){: .image}

Ya nos hemos hecho con Wordpress y tenemos acceso de administrador, así que pasamos a la tercer parte de la guía.

>Nota: Revisa el sitio por si encuentras algo interesante....como wallpapers ;)

## Tercer Parte: Haciendose con el servidor.

Vamos por esa key!!!!. Ahora el paso a seguir es lograr obtener una shell, para esto hay diversos métodos, cargar una web shell, cargar un plugin malicioso a Wordpress, etc.

Bueno si revisaste el sitio en busca de medios, páginas publicadas, etc, habrás notado un usuario diferente de elliot,
y que, no era parte del diccionario.

![Captura 15](/assets/img/mrRobot15.png){: .image}

Haciendo uso de Hydra nuevamente tenemos que en el diccionario estaba la contraseña de este usuario.

{% highlight bash %}
$ hydra 192.168.56.102 http-form-post "/wp-login.php:log=^USER^&pwd=^PASS^
:incorrect" -l mich05654 -P sorted.dic
{% endhighlight %}

![](/assets/img/mrRobot16.png){: .image}

> Another key?

Que quieren decirnos acá? EL siguiente paso será comprometer WordPress, para esto usaré MetaSploit FrameWork, si lo deseas puedes crear o usar una shell por medio de la adición de un plugin, etc. Veamos como será usando este Framework.

![Captura 17](/assets/img/mrRobot17.png){: .image}

Y buscamos el módulo asociado a la shell Wordpress.

![Captura 18](/assets/img/mrRobot18.png){: .image}

Cuyas opciones son las siguientes:

![Captura 19](/assets/img/mrRobot19.png){: .image}

Completamos con nuestros datos, obtenidos en los análisis anteriores, que se resumen a continuación:

> PASSWORD: ER28-0652
>
> USERNAME: elliot
>
> RHOST:  IP(192.168.56.102 en mi caso)
>
> TARGETURI: /
>
>VIRTUALHOST: www.mr-robot.com
>

Como PAYLOAD Tenemos las siguientes opciones:

![Captura 20](/assets/img/mrRobot20.png){: .image}

Esta vez usaremos **php/meterpreter/reverse_tcp**, por lo que además se debe especificar las opciones del PAYLOAD como sigue:

![Captura 21](/assets/img/mrRobot21.png){: .image}

Donde se debe especificar el LHOST o tu máquina, y el puerto que por defecto es 4444, una vez realizado lo ejecutamos y PUFFF!!!!

![Captura 22](/assets/img/mrRobot22.png){: .image}

Ya estamos en el servidor, a buscar la otra key....Navegando por las carpetas del servidor he encontrado un archivo de texto llamado **you-will-never-guess-this-file-name.txt**
el cual no es de mucha utilidad que digamos...Así que a seguir buscando, me pregunto cual será el usuario de la máquina......Revisar las carpetas es una buena idea, pero para acotar el resumen un tanto, iré directo al home, para descubrir el usuario, aunque estaŕas pensando que hay otros métodos para esto.

![Captura 23](/assets/img/mrRobot23.png){: .image}

Ok el usuario robot....veamos que contiene su carpeta.

![Captura 24](/assets/img/mrRobot24.png){: .image}

Creo que tendremos otra KEY!!! Como podemos ver los permisos de la key número 2 son limitados, por lo que no podremos acceder directamente, ahora bien sí se puede ver la password en MD5 así que nos haremos con el usuario robot ;) .

Usaremos la herramienta findmyhash para encontrar la contraseña:

{% highlight bash %}
findmyhash MD5 -h c3fcd3d76192e4007dfb496cca67e13b
{% endhighlight %}

![Captura 25](/assets/img/mrRobot25.png){: .image}

Con lo cual tenemos que la contraseña es **abcdefghijklmnopqrstuvwxyz**, ahora podremos iniciar la sesión en la máquina virtual, para esto en meterpreter usamos el comando
shell, pero nos afrontamos a un "problema" no poseemos una tty por tanto no podremos "usar" el sistema a nuestro antojo, lo que podemos solucionar realizando un spawn de una shell, para lo cual tenemos diversos métodos, uno de ellos es utilizando python, para lo cual debemos usar lo siguiente una vez estando en la shell:

{% highlight bash %}
python -c 'import pty; pty.spawn("/bin/sh")'
{% endhighlight %}

Luego podremos hacer un su robot, para ingresar como el usuario, y obtener la segunda key, la cual es:

> 822c73956184f694993bede3eb39f959

Genial!!! sólo nos falta una key y este CTF estaŕa completo!!!.
Habrá que hacerse con la cuenta **root**

Una de las maneras es utilizar el comando

{% highlight bash %}
find / -perm -4000 -uid 0 -type f 2>/dev/null
{% endhighlight %}

Lo que nos dará posibles vias de acceso a privilegios de administrador. Acá la creatividad es la que te indicará el camino a seguir, por ejemplo podemos ver la versión del kernel y utilizar una vulnerabilidad para escalar privilegios, etc. Ahora prestaremos atención a lo retornado por el comando antes mencionado.

![Captura 28](/assets/img/mrRobot28.png){: .image}

La aplicación que nos llevará a root es nmap, si lo siento no está robot en sudoers xD. Bueno hagamos que nmap nos ayude una vez más pero esta vez a escalar privilegios. Para esto ejecutemos nmap en modo interactivo.

{% highlight bash %}
nmap --interactive
{% endhighlight %}

Ahora usamos lo siguiente para cargar la shell, que debiería poseer permisos de administrador...

![Captura 29](/assets/img/mrRobot29.png){: .image}

Tenemos privilegios de administrador !!!!! Ok ahora vamos a buscar esa última key, primero a revisar el directorio /root en busca de algún hint.

{% highlight bash %}
# ls
firstboot_done  key-3-of-3.txt
{% endhighlight %}

Bueno con la máquina completamente comprometida veamos la tercer key:

>04787ddef27c3dee1ee161b21670b4e4

Y así hemos terminado el CTF!!. Espero hayan disfrutado tanto como yo resolviendolo, hasta la próxima!.

<a href="{{ 'ctf.html' | absolute_url }}" class="button big">Volver</a>

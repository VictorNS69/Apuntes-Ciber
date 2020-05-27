# Hacking Labs

Notas realizadas en el curso Online de [Ethical Hacker Fundamentals](https://www.pue.es/cursos/ec-council/ethical-hacker-fundamentals)

Se dividirá en 3 laboratorios:

- [Lab 1. Descubrir el perímetro de una red corporativa](#lab1-descubrir-el-perímetro-de-una-red-corporativa)
- [Lab 2. Ataques a los sistemas de autentificación corporativos](#lab2-ataques-a-los-sistemas-de-autentificación-corporativos)
- [Lab 3. Robo de las carpetas compartidas de una entidad mediante técnicas man-in-the-middle](#lab3-robo-de-las-carpetas-compartidas-de-una-entidad-mediante-técnicas-man-in-the-middle)
- [Lab 4. Conseguir privilegios en una máquina](#lab4-conseguir-privilegios-en-una-máquina)

Fecha: 27/05/20

## Lab1. Descubrir el perímetro de una red corporativa

### Encontrar objetivos en [infocif](http://www.infocif.es/)

Sirve para acceder a la información más reciente y completa de las empresas

### Información en RIPE

Investigar con [RIPE](https://ftp.ripe.net/ripe/dbase/ripe.db.gz) y buscar en la base de datos con `zcat ripe.db.gz | grep -i <key> -B 3` donde _"key"_ es lo que buscamos (como "Ayuntamiento") y _"-B 3"_ son líneas previas (nos devolverá el rango de IPs).

Por último, se puede verificar a quién pertenece la IP, haciendo `whois IP`, donde la IP es la IP en cuestión.

### Información en el mail

Podemos obtener alguna máquina (IP) mirando la cabecera de un email. Se puede usar la herramienta [mxtoolbox](https://mxtoolbox.com/EmailHeaders.aspx) para un análisis de cabezera más sencilla.

También se puede observar los campos SPF en su DNS mediante dig, con `dig TXT <dnsdomain>`.

### Información en el DNS

Usamos la herramienta **fierece** que partiendo de un dominio, hace fuerza bruta. El comando es `fierce -dns <dnsdomain>`. Otra herramienta para hacer esto, es [dnsdumpster](https://dnsdumpster.com/).

Con [Canary Tokens](https://canarytokens.org/generate) podemos generar un "canario" que te avise cada vez que alguien resuelve tu hostname.

### Información mediante escaneadores

#### Escaneo con [masscan](https://installlion.com/kali/kali/main/m/masscan/install/index.html)

```bash
sudo masscan -p 1-65535 <firstIP@>-<lastIP@> --rate=100000 -oB <outputfilename>
```

```bash
masscan --readscan <masscan-file>
```

#### Escaneo con nmap

```bash
nmap -n -p <know-ports> -sV <targetIP@>
```

```bash
grep -i found <fierce-file> | cut -d "(" -f 2|sed 's/)//'| sort | uniq > <outputfilename>
```

#### Escaneo con [shodan](https://www.shodan.io/)

```bash
shodan host <targetIP@>
```

```bash
cat <nmap-file> | while read line; do shodan host $line; done
```

## Lab2. Ataques a los sistemas de autentificación corporativos

### Obtención de usuarios

Se suelen buscar cuentas de correo en [hunter.io](https://hunter.io/)

Se pueden utilizar herramientas como [crosslinked](https://github.com/m8r0wn/CrossLinked) para descargar de linkedin y crear una base de datos de correos plausibles.

```bash
# {first}.{last}@pue.es se ha obtenido mirando en hunter.io
python3 ./crosslinked.py -f '{first}.{last}@pue.es' "PUE - Proyecto Universidad Empresa"
```

Después pasamos `sed` para quitar carácteres que sean problemáticos.

```bash
sed 'y/áÁàÀãÃâ éÉêÊíÍóÓõÕôÔúÚñÑçÇªº/aAaAaAaAeEeEiIoOoOoOuUnNcCao/' names.txt | sort | uniq > names2.txt
```

### Búsqueda de usuarios comprometidos

Se miran los correos obtenidos en [haveibeenpwned](https://haveibeenpwned.com/).

Los que den positivo, se pueden buscar en bases de datos de [leaks](https://gist.github.com/scottlinux/9a3b11257ac575e4f71de811322ce6b3).

```bash
cat names2.txt | while read line; do (echo "Buscando... $line" & ./query.sh $line); done
```

### Engaños por phishing

Realización de campañas de phishing mediante [gophish](https://getgophish.com/)

Enviamos correos falseando el remitente, por ejemplo con [Emkei](https://emkei.cz/)

### Fuerza bruta

#### Fuerza bruta ssh con **hydra**

```bash
hydra -vVd -l <user> -P <wordlist> -t 4 <destiny> ssh
```

#### Evaluar el fichero de **auth.log** o un **htaccess** de un servidor en producción

- Listado de usuarios:

```bash
grep -i "Failed password for invalid user" auth.log | cut -d " " -f 11 | sort | uniq

```

- Listado de IP que han forzado root:

```bash
grep -i "Failed password for root from" auth.log | cut -d " " -f 11 | uniq | sed '/Failed/d' | sort | uniq
```

#### Utilizar mecanismos preventivos como **fail2ban** o alertas de [GreyNoise](https://viz.greynoise.io/)

## Lab3. Robo de las carpetas compartidas de una entidad mediante técnicas man-in-the-middle

### Mirando el proceso **lsass** (windows)

Volcado del _lsass_ desde _Cain_ o software equivalente.

Obtención de contraseñas que fluyen por l a red con [Responder](https://github.com/SpiderLabs/Responder)

```bash
sudo python2 ./Responder.py -I <ethInterface> -wrfv
```

### Crack de un **hash ntlm**

Mediante un ataque al _lsass_ con [john](https://www.openwall.com/john/)

```bash
john <file>
```

Envio a [onlinehashcrack](https://www.onlinehashcrack.com/) o [gpuhash](https://gpuhash.me/).

### PitM (ARP poisoning) entre cliente y servidor SMB

Iniciamos un ARP poisoning y DNS Spoofing con **Ettercap**

```bash
sudo ettercap -i <ethInterface> -T -M arp:remote -P dns_spoof /<IP1>// /<IP2-start,IP-end>//
```

Con esto, podremos capturar las llaves NTLM y construir un fichero crack que es el que se le entrega a **[hashcat](https://hashcat.net/hashcat/)**.

```bash
sudo hashcat -m 5600 <file> <wordlist> --force
```

### PitM (fake DHCP) entre cliente y servidor SMB

Iniciamos un servicio DHCP que falsea el _gateway_ y el servidor DNS

```bash
sudo ettercap -i <ethInterface> -T -M dhcp:<IP1-start-IP1-end/<IP-gateway>/<IP2>
```

## Lab4. Conseguir privilegios en una máquina

Primero hay que encontrar la IP de la máquina

```bash
nmap -sn <ip/mask>
```

encontrar todos los puertos vulnerables

```bash
nmap -sS -n -v -p- <IP>
```

Buscar más información de esos puertos encontrados (en este caso 22 y 8000)

```bash
nmap -sC -sV -v -n -p22-8000 <IP>
```

Luego, miramos los posibles exploits bien en web o con `searchsploit <term>`.

Se pueden intertar a usar los diversos scripts encontrados con `searchsploit`.

También se puede usar fuerza bruta, como _hydra_.

Si por ejemplo, vemos que se aloja una página web, podemos intentar abirla e inspecionarla (con la consola del navegador).

Con `whatweb <IP>` podemos obtener cierta información sobre la web y las herramientas utilizadas en la máquina.

Lo siguiente, hacemos un ataque de diccionario para descubrir los posibles directorios. Hay diversas herramientas. Una de ellas es _gobuster_, que la podemos ejecutar con `gobuster dir -u <web> -w <wordlist> -x <extensions>`. Si addemas hacemos `| grep -v 403` devolverá todas las respuestas que no sean "403".

Podemos investigar con los resultados obtenidos, mirando las páginas webs, los directorios y mirando el código fuente de cada recurso encontrado, con la finalidad de encontrar material que sea relevante.

Supongamos que, hemos encontrado en cierto directorio, una vulnerabilidad en la configuración _php_ donde va a interpretar todo lo que se le pasa como parámetro en la petición HTTP. Podemos usar dicho parámetro para imprimir, por ejemplo el fichero _/etc/passwd_.

### Upgradear el terminal

\#TODO
"gtfobin web" para ver que se puede hacer con los binarios
comando "nc" invsetigar

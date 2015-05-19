# Análisis de la migración a Cloudstack

Se necesita migrar 7 servidores, 1 firewall, 3 VMs Windows. 
Además hay que migrar también herramientas de BBDD y aplicaciones de intranet (Java)

* Material necesario para la migración
  
  1 Servidor para Host2 (Cliente1)

  2 Servidores pequeños para GlusterFS

  1 tarjeta de red de perfil bajo para la máquina router (Nueva red 192.168.8.0/24)

  1 switch pequeño de cuatro/cinco puertos para la interconexión de los tres nuevos servidores
  
* Organización de la migración
------------------------------

  VM -> Servidor de correo y usuarios (Hay que saber si es Sendmail, Postfix u otro)

  VM -> Servidor Samba y usuarios

  VM -> Servidor OpenVPN

  VM -> Servidor Web Intranet

  VM -> Servidor Web Externo

  VM -> Servidor BBDD PostgreSQL

  VM -> Servidor Backup (dependiendo de como esté configurado el Backup)

  VM -> Firewall

  VM -> Herramientas de BBDD y Aplicaciones Intranet (Java)

  3 VMs -> Sistemas Windows

En total necesitamos 12 máquinas virtuales que correrían en un cluster de Cloudstack.
Para ello necesitamos crear una nueva Zona en Cloudstack con sus propias redes. Como 
ahora podemos crear las vxlan que necesitemos, crearíamos nuevas vxlan sobre la red física 
para Cliente1, de ese modo Dart y Cliente1 no deberían verse entre ellos aunque las dos infraestructuras 
estén gobernadas por el mismo Management.
El desarrollo de la operación de la migración sería:

* **Redes**

1) Creación de las redes vxlan y sus bridges sobre la red física de Dart.

    Red física del router -> vxlan2 -> pub1

        192.168.8.1  ->  ->    -> 192.168.12.1

    Red física del router -> vxlan3 -> mng1

        192.168.8.1  ->  ->    -> 192.168.13.1 mtu 4352

2) Redes de Host2 (Servidor que albergará todas la VMs de la migración)

    eth0 -> 192.168.8.5 (Invisible) mtu 4402

    eth0 -> vxlan2 -> pub1 Sin IP mtu 1500

    eth0 -> vxlan3 -> mng1 192.168.13.2 mtu 4352
    
* **Cloudstack**  

3) Creación de la Zona Cliente1, Pod Cliente1, Cluster Cliente1

    Dominio: cliente1

4) Configurar las redes de Cloudstack

    Interfaz pub1 -> Public (VLAN) editada como pub1

    Interfaz mng1 -> Management y Storage (VLAN) editadas como mng0

    Interfaz eth0 -> Guest (VXLAN) editada como eth0
    
5) Configuración de los distintos tráficos de red

    * Public -> GW 192.168.12.1 Netmask 255.255.255.0 VLAN/VNI -- Start IP 192.168.12.11 End IP 192.168.12.250

    * Pod -> GW 192.168.13.1 Netmask 255.255.255.0 VLAN/VNI -- Start IP 192.168.13.41 End IP 192.168.13.61

    * Guest -> VLAN range 110-200

    * Storage -> GW 192.168.13.1 Netmask 255.255.255.0 VLAN/VNI -- Start IP 192.168.13.71 End IP 192.168.13.91

* Storage

Una buena opción para el storage tanto para el primary como el secondary sería
la puesta en marcha de dos pequeños servidores GlusterFS con los volúmenes distribuidos y replicados, 
y con uno de ellos con el 51% de la quota.

Se crearán dos volúmenes (primary2 y secondary2) en los dos nuevos servidores GlusterFS.

6) Configuración de Primary Storage

    Nombre: Primary2

    Ambito: Cluster

    Protocolo: Gluster

    Servidor: storage2.cliente1

    Path: primary2
    
7) Configuración de Secondary Storage

    Nombre: Secondary2

    Protocolo: NFS

    Server: storage2.cliente1

    Path: /secondary2

Una vez tengamos configurado Cloudstack, pasaríamos a la descarga de las templates 
necesarias para la creación de las VMs. En principio todos los servidores de 
las VMs pueden correr sobre CentOS 7 por lo que solo tendríamos que descargar esta template.
   
**Hasta aquí si todo va bien, estas configuraciones no deberían llevarnos más de 24 horas (3 jornadas) 
puesto que ya tenemos preparado el Management y la creación de las redes es un trabajo que ya sabemos hacer, 
al igual que la instalación de GlusterFS y los volúmenes en los nuevos servidores.** 

8) Comienzo de la migración de servidores

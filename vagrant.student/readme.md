# 1. Configuración de los Servidores DNS

En este apartado configuramos los nombres de los servidores en Vagrant. La idea es que `atlas` y `ceo` tengan sus respectivos nombres de host bien definidos desde el inicio. No es complicado, pero hay que hacerlo bien para evitar problemas después.

## 1️.1 Configurar los nombres de los servidores en Vagrant

Para que cada servidor tenga su nombre de host correcto, editamos el archivo `Vagrantfile` y añadimos lo siguiente:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "256"
    vb.linked_clone = true
  end

  # Actualizar SO
  config.vm.provision "update", type: "shell", inline: <<-SHELL
    apt-get update 
  SHELL

  ######################################################################
  # Servidor 'atlas'
  ######################################################################
  config.vm.define "atlas" do |atlas|
    atlas.vm.hostname = "atlas"  # Asignamos el nombre de host
    atlas.vm.network "private_network", ip: "192.168.56.10"  # IP fija

    atlas.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end

  ######################################################################
  # Servidor 'ceo'
  ######################################################################
  config.vm.define "ceo" do |ceo|
    ceo.vm.hostname = "ceo"  # Asignamos el nombre de host
    ceo.vm.network "private_network", ip: "192.168.56.11"  # IP fija

    ceo.vm.provision "bind9-install", type: "shell", inline: <<-SHELL
        apt-get install -y bind9 bind9-utils bind9-doc
    SHELL
  end
end
```

## 1.2️ Iniciar las máquinas virtuales

Después de modificar el Vagrantfile, ejecutamos:

```bash
vagrant up
```

Si ya estaban encendidas, mejor reiniciarlas para que cojan bien los cambios:

```bash
vagrant reload --provision
```

## 1.3 Comprobar que los nombres están bien asignados

Para asegurarnos de que cada máquina tiene su nombre de host correcto, nos conectamos a cada una y ejecutamos:

```bash
vagrant ssh atlas #o ceo
hostname
exit
```

Si en cada caso aparece el nombre correcto (atlas o ceo), significa que todo está bien configurado.

# 2. Configurar la dirección IP de atlas y ceo
En este paso, se han asignado las direcciones IP a los servidores:

    Atlas: 192.168.56.10
    Ceo: 192.168.56.11

Para asegurarnos de que todo está bien, hemos comprobado las IPs con:

```bash
vagrant ssh atlas
ip a | grep 192.168.56
exit
```

```bash
vagrant ssh ceo
ip a | grep 192.168.56
exit
```
Si aparecen las IPs correctas, significa que la configuración está bien hecha.

# 3. Configurar la zona DNS olimpo.test
En este apartado configuramos la zona DNS olimpo.test para que atlas sea el servidor primario y ceo el secundario. Esto permitirá que atlas gestione los registros y que ceo reciba las actualizaciones de la zona cuando haya cambios.

## 3.1 Configurar las zonas en los servidores
### 3.1.1 En atlas
Editamos el archivo de configuración local de BIND:

```bash
sudo nano /etc/bind/named.conf.local
```

Añadimos:

```bash
zone "olimpo.test" {
    type master;
    file "/etc/bind/db.olimpo";
    allow-transfer { 192.168.56.11; };
    notify yes;
};

zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.olimpo.rev";
    allow-transfer { 192.168.56.11; };
    notify yes;
};
```
Guardamos y cerramos.

### 3.1.2 En ceo
Abrimos el mismo archivo:

```bash

sudo nano /etc/bind/named.conf.local
```

Añadimos:

```bash
zone "olimpo.test" {
    type slave;
    file "/var/cache/bind/db.olimpo";
    masters { 192.168.56.10; };
};

zone "56.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.olimpo.rev";
    masters { 192.168.56.10; };
};
```

Guardamos y cerramos.

## 3.2 Crear los archivos de zona en atlas
### 3.2.1 Zona directa

```bash
sudo nano /etc/bind/db.olimpo
```

y añadimos:

```bash
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        1        ; Serial
        1200     ; Refresh
        900      ; Retry
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

@   IN  NS  atlas.olimpo.test.
atlas   IN  A   192.168.56.10
ceo     IN  A   192.168.56.11
```

### 3.2.2 Zona inversa
```bash
sudo nano /etc/bind/db.olimpo.rev
```

```bash
$TTL 86400
@   IN  SOA atlas. hefestos.olimpo.test. (
        1        ; Serial
        1200     ; Refresh
        900      ; Retry
        2419200  ; Expire
        86400    ; Negative Cache TTL
)

@   IN  NS  atlas.olimpo.test.
10  IN  PTR atlas.olimpo.test.
11  IN  PTR ceo.olimpo.test.
```
## 3.3 Reiniciar BIND y comprobar
### 3.3.1 Reiniciar BIND

En atlas:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

En ceo:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

### 3.3.2 Verificar la transferencia de zona

En ceo:

```bash
ls -l /var/cache/bind/
```

Si aparecen los archivos db.olimpo y db.olimpo.rev, significa que la transferencia ha funcionado.

# 4. Configurar el responsable de los servidores y los tiempos de refresco
En este paso, nos aseguramos de que el responsable de los servidores sea hefestos@olimpo.test y configuramos los tiempos de refresco y reintento en los archivos de zona.

## 4.1 Modificar los archivos de zona en atlas
Abrimos el archivo de zona directa:

```bash
sudo nano /etc/bind/db.olimpo
```

Nos aseguramos de que el bloque SOA tenga los valores correctos:

```bash
@   IN  SOA atlas. hefestos.olimpo.test. (
        2        ; Serial
        1200     ; Refresh (20 min)
        900      ; Retry (15 min)
        2419200  ; Expire
        86400    ; Negative Cache TTL
)
```

Hacemos lo mismo en la zona inversa:

```bash
sudo nano /etc/bind/db.olimpo.rev
```

Nos aseguramos de actualizar el número de serie y guardamos los cambios.

## 4.2 Recargar BIND y verificar
Después de modificar los archivos, reiniciamos BIND en atlas:

```bash
sudo systemctl restart bind9
```

Para asegurarnos de que ceo recibe la actualización, ejecutamos en ceo:

```bash
sudo rndc retransfer olimpo.test
sudo rndc reload
```

Luego verificamos si la actualización se ha propagado correctamente:

```bash
dig @192.168.56.11 SOA olimpo.test
```

Si el número de serie mostrado coincide con el de atlas, la configuración se ha aplicado correctamente.
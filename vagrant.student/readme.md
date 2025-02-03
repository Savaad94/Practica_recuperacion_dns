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
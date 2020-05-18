+++
autores = ["pericles jr"]
title = "Box local no Vagrant com KVM(Libvirt)"
date = 2020-05-07T16:18:36-03:00
description = "Crie localmente suas próprias Boxes no Vagrant com KVM(libvirt)"
draft =  false
tags = [
    "Vagrant",
    "KVM",
    "Libvirt",
]
+++

Nesse post irei mostrar como criar suas *Boxes* localmente no *Vagrant* com o *KVM(libvirt)*.
<!--commentarys-->

## Introdução

O *<cite>Vagrant[^1]</cite>* é uma interessante ferramenta desenvolvida na linguagem *Ruby*, mantida pela empresa *Hashicorp* e licenciada sob a *MIT License*. Sua principal função é o gerenciamento de máquinas virtuais em um único fluxo de trabalho. Resumidamente, você usa um *<cite>provider[^2]</cite>*, *VirtualBox* ou *<cite>KVM(libvirt)[^3]</cite>* por exemplo, baixa uma *<cite>Box[^4]</cite>* no *<cite>Vagrant Cloud[^5]</cite>* e define o estado (ip, hostname, memória e etc) que você deseja nela em arquivo de configuração chamado *<cite>Vagrantfile[^6]</cite>*.

[^1]: https://www.vagrantup.com/intro/index.html
[^2]: https://www.vagrantup.com/docs/providers/
[^3]: https://www.linux-kvm.org/page/Main_Page
[^4]: https://www.vagrantup.com/docs/boxes.html
[^5]: https://app.vagrantup.com/boxes/search
[^6]: https://www.vagrantup.com/docs/vagrantfile/

## Motivação

Se houver a necessidade de um ambiente mais customizado? Por exemplo, com um diferente particionamento no disco ou até mesmo outros tipos de *filesystems*. Pesquisando na *<cite>interwebs[^7]</cite>*, descobri que no *Vagrant* existe a possibilidade de gerar localmente a sua prórpia *box* e usá-la no *hypervisor KVM(libvirt)*.

[^7]: https://leyhline.github.io/2019/02/16/creating-a-vagrant-base-box/ 


## Setup do ambiente

Pacotes instalados:

* vagrant
* qemu-kvm
* libvirt-daemon

Sistema Operacional do *Host*:

* Debian Testing

Plugin do *Vagrant*:

* vagrant-libvirt

## Preparando o ambiente

Iremos dividir em duas etapas o procedimento para criação das *boxes* localmente. A primeira etapa será para criação e configuração da máquina virtual que será usada como *template* para a *box*. Já a segunda etapa será a preparação do ambiente para o *Vagrant* convertendo o disco da VM em uma *image* para a *box*.

### 1ª Etapa 

#### Criação da máquina virtual

1.1 - Crie uma VM ao seu critério (particionamento, *filesystem* e etc), vou adotar que será instalado um *Ubuntu* 18.04.4 usando o *KVM(Libvirt)*.

1.2 - Crie um novo usuário para o *Vagrant* na VM com o *Ubuntu* 18.04.4:

```
# adduser --disabled-password --gecos "Vagrant User" vagrant
```

1.3 - Habilite o usuário recém criado na VM para usar o sudo sem senha:

```
# visudo -f /etc/sudoers.d/vagrant
```

Acrescente esse conteúdo ao arquivo e salve no editor padrão que `visudo` irá abrir:

```
vagrant ALL=(ALL) NOPASSWD:ALL
```

1.4 - Caso não tenha na VM, instale o serviço de SSH:

```
# apt-get install openssh-server
```

1.5 - Configure a autenticação via par de chaves (*SSH keys*) no SSH

O *Vagrant* por padrão usa um par de chaves inseguras (*<cite>insecure keypair[^8]</cite>*) para o acesso *SSH* às VMs, portanto para termos a automação com mais segurança, iremos gerar nossas próprias chaves e configurar o acesso remoto neste serviço para utilizá-las.

[^8]: https://github.com/hashicorp/vagrant/tree/master/keys 

Gere o par de chaves da seguinte maneira:

```
$ ssh-keygen -t rsa -b 2048 -q -f $HOME/.ssh/vagrant/vagrant_key -C 'vagrant' -N ''
```

Agora iremos criar o diretório `.ssh` no `home` do usuário `vagrant` para o arquivo `authorized_keys`, dar as devidas permissões e copiar a chave criada no passo anterior no *Host* (sua estação de trabalho) para o diretório criado na VM:

```
ssh -l root IP-DA-VM \
"sudo -u vagrant mkdir -p /home/vagrant/.ssh; \
sudo -u vagrant touch /home/vagrant/.ssh/authorized_keys; \
chmod 700 /home/vagrant/.ssh; \
chmod 600 /home/vagrant/.ssh/authorized_keys; \
sh -c 'cat >> /home/vagrant/.ssh/authorized_keys'" < ~/.ssh/vagrant/vagrant_key.pub
```

OBS: Lembre-se substituir `IP-DA-VM` pelo **endereço IP da VM** no comando acima, além de ter o *SSH* com acesso habitado para o usuário `root`, o qual será desabilitado posteriormente.

1.6 - Configure o serviço do *SSH* na VM

Edite o arquivo `/etc/ssh/sshd_config` usando o editor de sua preferência no console, vim ou nano por exemplo:

```
# vim /etc/ssh/sshd_config
```

Revisando/Alterando os seguintes parâmetros conforme abaixo:

```
PermitRootLogin no

PubKeyAuthentication yes

AuthorizedKeysFile %h/.ssh/authorized_keys

PermitEmptyPasswords no

PasswordAuthentication no

UseDNS no
```

Salve e feche o arquivo.

OBS: Não precisa reiniciar o serviço, pois a VM será desligada posteriormente.

1.7 - Opcional… se quiser adiantar seu lado... instale alguns programas básicos na VM de sua preferência:


```
# apt-get -y install tcpdump strace ethstatus tree
```

OBS: Para esse exemplo do *Ubuntu* 18.04.4, será necessário fazer um *workaround* para o problema das nomenclaturas da interface de rede no *Vagrant* com *KVM(Libvirt)*.

Configure o parâmetro abaixo no `/etc/default/grub`

De:


> **GRUB_CMDLINE_LINUX=""**

Para:

> **GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"**


E execute o comando para persistir a configuração no próximo boot:

```
# update-grub
```

Por último, ajuste o arquivo do *<cite>netplan[^9]</cite>*, o utilitário de configuração de rede do sistema, com o comando `sed`:

[^9]: https://netplan.io/


```
# sed -i.BKP 's/enp1s0/eth0/g' /etc/netplan/50-cloud-init.yaml
```

Ele substituirá a string `enp1s0` para `eth0`, criando uma cópia de *backup* sem a alteração no arquivo `50-cloud-init.yaml.BKP`.

1.8 - Finalizado tudo desligue a VM.


### 2ª Etapa 

#### Configuração do ambiente para o *Vagrant*

A segunda parte consistirá em converter a *image*(disco) da VM da seção anterior para ser uma *box* no *Vagrant* e configurar os arquivos dele.

2.1 - Crie um diretório temporário na sua estação de trabalho:

```
# mkdir /tmp/box && cd /tmp/box

```

2.2 - Baixe o script do repositório *<cite>vagrant-libvirt[^10]</cite>*:


```
$ wget https://raw.githubusercontent.com/vagrant-libvirt/vagrant-libvirt/master/tools/create_box.sh
```

[^10]: https://github.com/vagrant-libvirt/vagrant-libvirt/

2.3 - Copie o disco da VM recém instalada localizado no diretório *padrão* do *KVM(libvirt)* renomeando-o:

```
# cp /var/lib/libvirt/images/NOME_DO_DISCO.qcow2 ubuntu1804.qcow2
```

2.4 - Execute o script:

```
# bash create_box.sh ubuntu1804.qcow2
```

Obterá como saída

```
==> Creating box, tarring and gzipping
./metadata.json
./Vagrantfile
./box.img
Total bytes written: 345384960 (330MiB, 168MiB/s)
==> ubuntu1804.box created
==> You can now add the box:
==>   'vagrant box add ubuntu1804.box --name ubuntu1804'
```

Após a execução será gerado o arquivo, nesse caso o **ubuntu1804.box**.

2.5 - Importe o arquivo gerado anteriormente:

```
$ vagrant box add ubuntu1804.box --name ubuntu1804
```

Obterá como saída:

```
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'ubuntu1804' (v0) for provider: 
    box: Unpacking necessary files from: file:///tmp/box/ubuntu1804.box
==> box: Successfully added box 'ubuntu1804' (v0) for 'libvirt'!
```

2.6 - Para conferir:

```
$ vagrant box list
ubuntu1804 (libvirt, 0)
```

2.7 - Para remover:

```
$ vagrant box remove ubuntu1804
Removing box 'ubuntu1804' (v0) with provider 'libvirt'...
Vagrant-libvirt plugin removed box only from you LOCAL ~/.vagrant/boxes directory
From libvirt storage pool you have to delete image manually(virsh, virt-manager or by any other tool)
```

2.8 - Crie uma pasta para seus projetos e dentro dela crie um *Vagrantfile*:

```
$ mkdir mylabs && cd mylabs
```

Para criar o arquivo:

```
$ cat > Vagrantfile <<EOF
Vagrant.configure("2") do |config|
	config.vm.provider :libvirt do |libvirt|
     	libvirt.host = "localhost"
     	libvirt.uri = "qemu:///system"
     	libvirt.management_network_name = "default"
     	libvirt.management_network_address = "192.168.122.0/24"
	end

	config.ssh.insert_key = false
	config.ssh.private_key_path = ["~/.ssh/vagrant/vagrant_rsa_key"]

	config.vm.define :ubuntu1804 do |ubuntu1804|
 		ubuntu1804.vm.hostname = "vm-ubuntu1804"
  		ubuntu1804.vm.box = "ubuntu1804"
  		ubuntu1804.vm.provider :libvirt do |domain|
    		domain.memory = 512
    		domain.cpus = 1
  		end
 	end
end
EOF
```

OBS: Com essa configuração o Vagrant por padrão criará um compartilhamento *NFS* entre o diretório do projeto(mylabs) e a pasta /vagrant dentro da VM (*<cite>synced folders[^11]</cite>*), detalhe é necessário que o usuário local do seu sistema esteja também com o sudo configurado para o compartilhamento funcionar.

[^11]: https://www.vagrantup.com/docs/synced-folders/

Caso não queira, o `config.vm.define` deverá ficar assim

```
	config.vm.define :ubuntu1804 do |ubuntu1804|
 		ubuntu1804.vm.hostname = "vm-ubuntu1804"
 		ubuntu1804.vm.box = "ubuntu1804"
 		ubuntu1804.vm.synced_folder ".", "/vagrant", disabled: true
 		ubuntu1804.vm.provider :libvirt do |domain|
    		domain.memory = 512
    		domain.cpus = 1
  		end
 	end
```

Após terminar a edição do `Vagrantfile`, valide a sintaxe dele:

```
$ vagrant validate
Vagrantfile validated successfully.
```

2.9 - E finalmente subimos a VM com o comando

```
$ vagrant up
```

Para *<cite>Debugging[^12]</cite>*, existe a possibilidade de iniciar com a variável de ambiente `VAGRANT_LOG` escolhendo o nível de detalhamento do log, por exemplo:

[^12]: https://www.vagrantup.com/docs/other/debugging.html

```
$ VAGRANT_LOG=info vagrant up
```

2.10 - Para acessar a VM:

```
$ vagrant ssh
```

E pronto! Agora você pode acompanhar/colaborar com os *<cite>issues[^13]</cite>* também.

[^13]: https://github.com/vagrant-libvirt/vagrant-libvirt/issues

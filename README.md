# hassioserverlocal_opiz3
Servidor Local Home Assistant em Orange PI Zero 3

**1 - Atualizar sistema**

sudo su - \
sudo apt update && sudo apt full-upgrade -y

**Instalar Pacotes HASSIO**

apt install \
apparmor \
jq \
wget \
curl \
udisks2 \
libglib2.0-bin \
network-manager \
dbus \
lsb-release \
systemd-journal-remote -y

**Configurar Debian para CGroup1 (Debian 11 já vem com Cgroup2 porém HASSIO ainda está no Cgroup1)**

echo "extraargs=apparmor=1 security=apparmor" >> /boot/orangepiEnv.txt
sed -i -e "1 s/$/ systemd.unified_cgroup_hierarchy=0/" /boot/orangepiEnv.txt
update-initramfs -u

reboot

**Após reiniciar conferir se o sistema está no Cgroup correto:**

systemctl status apparmor.service

**Você deverá ver uma linha dizendo ativo em verde**

**Verificar se Cgroup esta em 1**

findmnt -lo source,target,fstype,options -t cgroup,cgroup2

**Você deverá ver muitas linhas com cgroup na coluna de origem**

**2 - Listar unidades de armazenamento do sistema**

sudo lsblk

Exemplo

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT

sda           8:0    0 232.9G  0 disk 

├─sda1        8:1    0     1M  0 part 

mmcblk0     179:0    0  59.5G  0 disk 

└─mmcblk0p1 179:1    0  58.9G  0 part /

zram0       252:0    0 492.1M  0 disk [SWAP]

zram1       252:1    0    50M  0 disk /var/log

zram2       252:2    0 492.1M  0 disk /tmp

**3 - Limpar SSD**

sudo fdisk /dev/sda

e2fsck /dev/sda1 -y 

sudo e2fsck /dev/sda1 -y	

Para OPI 3B – Limpar Emmc

NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT

mtdblock0     31:0    0    16M  0 disk

mmcblk0      179:0    0   233G  0 disk

├─mmcblk0p1  179:1    0     1G  0 part /boot

└─mmcblk0p2  179:2    0 229.6G  0 part /

mmcblk0boot0 179:32   0     4M  1 disk

mmcblk0boot1 179:64   0     4M  1 disk

zram0        254:0    0   1.9G  0 disk [SWAP]

zram1        254:1    0    50M  0 disk /var/log

sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.36.1).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
Command (m for help):
Digitar “m” para listar opções, depois deletar a partição selecionando “d”, depois mudar o tipo de partição para Linux codigo 83. Com comando “w” para sair e salvar.

4 - Copiar td do SD para o SSD (Obs.: mudar o nome mmcblk0, conforme aparecer para seu SD card)
sudo cat /dev/mmcblk0 > /dev/sda
sudo cat /dev/mmcblk1 > /dev/mmcblk0

5 - Desconect o SSD e conect novamente:
sudo tune2fs -U random /dev/mmcblk0p1

6 - Ver se o UUID é diferente do Cartão SD e do SSD, precisam ser diferentes:
sudo blkid
exemplo:
/dev/mmcblk1p1: UUID="9c0a441b-5aaf-4cf1-bec7-b457d3966cb6" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="e8ce0794-01"
/dev/zram0: UUID="66b180ff-2899-431b-b8b7-8347d9af85eb" TYPE="swap"
/dev/zram1: LABEL="log2ram" UUID="d87403e0-1d2f-4356-905d-9dfaba14b6ee" BLOCK_SIZE="4096" TYPE="ext4"
/dev/zram2: LABEL="tmp" UUID="9abe5f8e-49b7-41d6-839b-3d2e0aa3b9cd" BLOCK_SIZE="4096" TYPE="ext4"
/dev/sda1: UUID="5dd5b836-fb32-4678-908e-d23f7a028780" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="e8ce0794-01"

7 - Antes de reiniciar digite: 
sudo e2fsck -yf /dev/sda1
Isso evitará que você execute fsck na CLI na primeira vez que tentar inicializar.

8 - REiniciar 
sudo reboot
Listar partições:
mount
Aqui podemos ver que nossa partição raiz (/) está de fato em /dev/sda e não em /dev/mmcblk0. Sucesso!

Depois de reiniciar e verificar se você inicializou a partir do SSD, podemos redimensionar a partição do SSD para o mesmo tamanho.
Podemos usar um serviço integrado no Orange Pi para fazer isso. Use o seguinte comando:
sudo systemctl start orangepi-resize-filesystem.service
Agora o SSd está com tamanho real correto.

Instalar Docker e HA
Install DockerCE
curl -fsSL get.docker.com | sh

Install Home Assistant OS Agent
Download and install the latest version from https://github.com/home-assistant/os-agent/releases/latest. Look for aarch64.deb file. For instance:

wget https://github.com/home-assistant/os-agent/releases/download/1.5.1/os-agent_1.5.1_linux_aarch64.deb
https://github.com/home-assistant/os-agent/releases/download/1.6.0/os-agent_1.6.0_linux_aarch64.deb
dpkg -i os-agent_1.5.1_linux_aarch64.deb

https://github.com/home-assistant/os-agent/releases/download/1.6.0/os-agent_1.6.0_linux_aarch64.deb
dpkg -i os-agent_1.6.0_linux_aarch64.deb

Test instalation by running
gdbus introspect --system --dest io.hass.os --object-path /io/hass/os
Some results in JSON format should be returned

Install Home Assistant Supervised
wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
apt install ./homeassistant-supervised.deb

When prompted, select qemuarm-64 machine type. I’m not sure that’s the best option, but it works.
Just wait until instalation is completed (it should take a few seconds). Some warnings are excpected, since this OS is a custom Debian build.

Try to access http://iphomeassistant:8123. It should work, if not use host IP. If it still doesn’t work reboot machine and try again. If it still doesn’t work, go back to step 1 and review everything.

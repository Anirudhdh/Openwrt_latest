Шить OpenWRT в Firmware Recovery Mode. Выключить. Включить с зажатым Reset(держать 15 сек). Загуржаем прошивку(https://openwrt.org/toh/zbt/we-826) через веб https://192.168.1.1.

Когда появляется пинг заходим по ssh без пароля(возможно будет старый пароль):

```
ssh root@192.168.1.1
```
Настраиваем консоль:

```bash
cat >> /root/.profile << __EOF__
PS1='\[\e[0;33m\]\u \h\[\e[m\] \[\e[\e[1;2m\]\w\[\e[m\]\[\e[0;33m\] \[\e[m\]\[\e[0;37m\]'
export EDITOR="nano"
alias l='ls -alFh'
alias ll='ls -alFh'
alias lll='ls -alFh'
alias cp='cp -i'
alias mv='mv -i'
alias shre='shred -uvz'
alias qr='qrencode -t ANSIUTF8 $*'
alias myip='wget -qO- http://ipecho.net/plain | xargs echo'
alias grep='grep --color=always'
cd /etc/config
__EOF__
source /root/.profile
```

Установить правильное время в /etc/profile/system и перезагрузиться:

```shell
option hostname 'UF0'
option timezone 'MSK-3'
option zonename 'Europe/Moscow'
```
```/etc/init.d/system restart```

Устанавливаем софт который понадобится для переноса рута на флешку:

```bash
apk --update-cache add kmod-usb-storage kmod-usb-ohci kmod-usb2 kmod-usb3 kmod-fs-ext4 kmod-fs-vfat kmod-usb-serial-option kmod-nls-cp437 kmod-nls-iso8859-1 kmod-fs-f2fs f2fs-tools block-mount kmod-usb-uhci e2fsprogs gdisk blkid usbutils nano unzip
```

Остальное устанавливаем **после** переноса рута на флешку:

```bash
echo 'https://github.com/4IceG/Modem-extras-apk/raw/refs/heads/main/myapk/packages.adb' >> /etc/apk/repositories.d/customfeeds.list && \
wget https://github.com/4IceG/Modem-extras-apk/raw/refs/heads/main/myapk/IceG-apkpub.pem -O /etc/apk/keys/IceG-apkpub.pem
apk update && \
apk del wpad-basic-mbedtls && \
apk --update-cache add wpad-openssl wireguard-tools minicom uqmi luci-proto-qmi luci-app-mwan3 luci-i18n-mwan3-ru mwan3 luci-app-sms-tool-js luci-app-modemdata nginx-ssl luci-ssl-openssl luci-ssl-nginx git openssh-client-utils openssl-util lsblk diffutils curl xray-core nping-ssl
```
Посмотрим, какие есть у вас разделы:

```
block info
```

Если потребуется размечаем флешку(gdisk'ом), создаем ФС на разделах. *sda1 - usb флешка, mmcblk0p1  - microsd флешка*:

```bash
mkfs.ext4 -F -L FLDRG_ORLAY /dev/mmcblk0p1
mkswap -L FLDRG_SWAP /dev/mmcblk0p2
#mkfs.ext4 -F -L FLDRG_FS /dev/mmcblk0p3
```

Копируем содержимое встроенного рут раздела на раздел на флешке:

```bash
mkdir -p /tmp/cproot
mount --bind /overlay /tmp/cproot
mount /dev/mmcblk0p1 /mnt
tar -C /tmp/cproot -cvf - . | tar -C /mnt -xf -	
umount /tmp/cproot /mnt
```

`reboot`

Включаем swap и создаем директорию для монтирования раздела для файлов на флешке:

```bash
swapon /dev/mmcblk0p2
mkdir /sdcard
service fstab disable
```

Редактируем /etc/config/fstab:

`echo '' > /etc/config/fstab`

```bash
config global
        option anon_swap '0'
        option anon_mount '0'
        option auto_swap '1'
        option auto_mount '1'
        option delay_root '5'
        option check_fs '0'

config swap
       option          label           'FLDRG_SWAP'
       option          enabled         '1'

config mount
        option          target          '/sdcard'
        option          label           'FLDRG_FS'
        option          enabled         '1'

config mount 'overlay'
        option          target          '/overlay'
        option          label           'FLDRG_ORLAY'
        option          enabled         '1'
```

```
reboot
```

Возможно, после перезагрузки прийдется повторить запись fstab и reboot.

```
ssh root@192.168.1.1
```

~~_*Вариант 2*_: Устанавливаем доп. софт. Клонируем реп с конфигами с git'a:~~

~~Для этого устанавливаем ssh-agent и добавляем приватный ключ от git репозитория. Сам ключ копируем в **~/.ssh/router_deploy_key**:~~

```bash
mkdir ~/.ssh/
```

~~Загрузить router_deploy_key(и config):~~

```bash
chmod 600 ~/.ssh/router_deploy_key
eval `ssh-agent -s`
ssh-add ~/.ssh/router_deploy_key
```

~~Добавляем ключ и алис в конфиг ssh агента ***~/.ssh/config***. Далее нужно будет заменить в origin url репозитория github.com на алис git-repo-1, что бы коммитить в реп:~~

```bash
Host git-repo-1
HostName github.com
IdentityFile ~/.ssh/router_deploy_key
```

~~Клонируем router git репозиторийво временную директорию и делаем слияние:~~

```
mkdir /tmp/config
git clone git@git-repo-1:Anirudhdh/openwrt.git /tmp/config
```

~~Далее.~~

~~Меняем владельца всех файлов на root, копируем конфиги в /etc/config:~~

```bash
chown -R root:root /tmp/config
cp -rp /tmp/config/* /etc/config
#cp -Rp /tmp/config/openwrt-main/* /etc/config
```

~~Переносим конфиги из бекапа.~~	

~~Устагавливаем пароль рута и перезагружаемся:~~

```
passwd
reboot
```

**Дополнительно**:

Меняем Apache на Nginx:

```bash
#apk --update-cache add nginx-ssl luci-ssl-openssl luci-ssl-nginx # nginx уже установлен
service uhttpd disable
service nginx enable
```

Меняем Dropbear на Openssh-сервер:

```
apk --update-cache add openssh-server
```

Делаем настройки в /etc/ssh/sshd_config:

```bash
Port 2552
PermitRootLogin yes
```

~~Добавляем публичные ключи клиентов в /root/.ssh/authorized_keys. Перезагружаемся.~~

~~Проверяем. Если все работает отключаем dropbear и перезагружаемся:~~

```bash
/etc/init.d/dropbear disable
reboot
```

Выключаем mwan3, если необходимо:

```bash
service mwan3 stop
service mwan3 disable
```

Установка и настройка EAP-TLS WIFI с Freeradius.

Install the FreeRADIUS packages:

```bash
opkg install freeradius3-default freeradius3-utils
```

```bash
openssl ecparam -out ecca.param -name secp384r1
openssl req -nodes -newkey ec:ecca.param -days 3650 -x509 -sha256 -keyout ecca.key -out ecca.crt
Common Name (e.g. server FQDN or YOUR name) []:routerca
openssl req -nodes -newkey ec:ecca.crt -days 3650 -sha256 -keyout serverec.key -out serverec.csr
Common Name (e.g. server FQDN or YOUR name) []:serverec


mkdir ./demoCA/
mkdir ./demoCA/newcerts
touch ./demoCA/index.txt
echo 01 > ./demoCA/serial

openssl ca -extensions v3_ca -days 3650 -out serverec.crt -in serverec.csr -cert ecca.crt -keyfile ecca.key
cat ecca.crt ecca.key > ca.pem
cat serverec.crt serverec.key > server.pem


openssl req -nodes -newkey ec:ecca.crt -days 3650 -sha256 -keyout mimi.key -out mimi.csr
Common Name (e.g. server FQDN or YOUR name) []:mimi


openssl ca -extensions v3_ca -days 3650 -out mimi.crt -in mimi.csr -cert ecca.crt -keyfile ecca.key
openssl pkcs12 -export -in mimi.crt -inkey mimi.key -out mimi.p12 -certfile ecca.crt
Enter Export Password:qwegflj0gwwg(htg209has09ghj02h09h2g

Don't forget to copy your self-signed ecca.crt file to your different clients that need a client certificate to authenticate.
ecca.crt только серт, без приватного ключа

/etc/freeradius3/mods-enabled/eap

default_eap_type = tls
ecdh_curve = "secp384r1" (!)

#my: section: eap {
#my: comment out 'md5' section
#	md5 {
#	}
#
#my: comment out 'leap' section
#	leap {
#	}
#
#my: comment out 'gtc' section
#	gtc {
#	...
#	}
#my: comment out 'EAP-TTLS' section
#	ttls {
#	...
#	}
#my: comment out 'EAP-PEAP' section
#	peap {
#	...
#	}
#my: comment out 'EAP-MSCHAPv2' section
#	mschapv2 {
#	...
#	}	

/etc/freeradius3/clients.conf
client localhost {
	. . . 
	ipaddr = 127.0.0.1
	secret = "jJdDjtCjgkfjfj11"
```



***Tips & Tricks***:

Передать файл на хост:

```bash
scp -O opwrt.2021.04.12.tgz root@router:/etc/config/opwrt.2021.04.12.tgz
```

Забрать файл с хоста:

```bash
scp -O root@UF0:/etc/config/FreeRadius3/certs/certsmi.tgz /home/one/certsmi.tgz
```

Траблшутинг dnscrypt-proxy:

```bash
dnscrypt-proxy -config dnscrypt-proxy.toml -check
ps|grep dns
logread|grep dnsmasq
ip rule list | awk '/lookup/ {print $NF}'
```

Обновление из консоли:

```bash
sysupgrade -F -n openwrt-19.07.7-ramips-mt7620-miwifi-mini-squashfs-sysupgrade.bin
```

Список подключенных хостов:

```bash
arp
iwinfo phy0-ap0 assoclist
ubus call system board
```

Установка xray:

```bash
apk --update-cache add xray-core
```

Сконфигурировать запуск сервиса в /etc/config/xray:

```
config xray 'enabled'
        option enabled '1'

config xray 'config'
        option confdir '/etc/xray'
        list conffiles '/etc/xray/config.json'
        option datadir '/usr/share/xray'
        option dialer ''
        option format 'json'
```

```
service xray start
service xray enable
```

Вход по ключу(токену):

```bash
gpg --export-ssh-key B6FACC66BC25F71B! > authkey.pub
ssh-copy-id -f -i authkey.pub root@10.0.0.1
```

Мусорный пакет для установки wg соединения в обход блокировок(Listen Port в config wireguard_wg1):

```
nping --udp --count 1 --data-length 16 --source-port 35464 --dest-port 52345 150.241.116.115;ifup wg1
```

```
option listen_port              '52345'
```

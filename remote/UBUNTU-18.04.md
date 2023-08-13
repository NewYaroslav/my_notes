# Инструкция по развертыванию сервера для алготрейдинга для ОС Ubuntu 18.04

Данная инстркукция подходит для работы с Ubuntu 18.04 x86-64

## Некоторые полезные команды

* Перезагрузка сервера

```
sudo systemctl reboot
```

или

```
sudo reboot now
```

* Смена пароля пользователя

Вы можете сменить свой пароль, когда захотите. Для этого вам не нужно особых прав суперпользователя, только знать свой текущий пароль. Просто откройте терминал и выполните утилиту passwd без параметров:

```
passwd
```

Сначала система запросит старый пароль. Дальше необходимо ввести новый пароль, повторно новый пароль - и готово, теперь он измеён. Он кодируетсятся с помощью необратимого шифрования и сохраняется в файле /etc/shadow Но заметьте, что вы не можете использовать здесь любой пароль. Система Linux заботится о том, чтобы пользователи выбирали достаточно сложные пароли. Если он будет очень коротким или будет содержать только цифры, вы не сможете его установить.

Общие требования для пароля такие: должен содержать от 6 до 8 символов, причём один или несколько из них должны относиться как минимум к двум из таких множеств:

Буквы нижнего регистра
Буквы верхнего регистра
Цифры от нуля до девяти
Знаки препинания и знак _

* Обновление браузера

```
sudo apt update && sudo apt upgrade -y
```

## Подключение к серверу

* Для подключения к серверу из среды Windows можно использовать [putty](https://www.putty.org/)
* После запуска putty указываем IP-адрес, открываем соединение.
* Если появляется всплывающее окно с предупреждением, нажимаем *да*
* В поле *login* указываем пользователя root, пароль указываем тот, который пришел на почтовый ящик от хостера, или который вы сами задали на сайте хостера. Пароль во время ввода в консоль pytty не видно, это нормально.
* Для копирования текста из консоли подходит сочетание клавиш *CTRL+V*

## Настройка VNC/XDRP и GUI по шагам

### 1. Установка GUI

```
$ sudo apt-get update
$ yes | sudo apt-get install --no-install-recommends xserver-xorg xserver-xorg-core xfonts-base xinit libgl1-mesa-dri x11-xserver-utils
```

### 2. Установим Xfce

Стандартная установка:
```
$ sudo apt-get update
$ yes | sudo apt-get install xfce4 xfce4-terminal
```

FAQ по xfce: https://wiki.xfce.org/ru/faq

### 3. Меняем пароль

Рекомендация от хостера, сменить стандартный пароль на свой.

```
sudo passwd
```

### 4. Создаем пользователя с ограниченными привилегиями

Для безопасности нужно создать нового пользователя с ограниченными привилегиями для повседневных задач

```
sudo adduser новое_имя_пользователя
sudo usermod -aG sudo новое_имя_пользователя
```

Теперь вы можете использовать команду sudo для выполнения задач с привилегиями администратора.


### 5. Настраиваем удаленный доступ

Чтобы нормально работал автологин сессии, нужно использовать *x11vnc*. 
Установка следующая:

```
sudo apt-get install x11vnc
sudo x11vnc -storepasswd ваш_пароль /etc/x11vnc.pass
sudo chmod ugo+r /etc/x11vnc.pass
sudo nano /lib/systemd/system/x11vnc.service

# пишем в файле для автозапуска

[Unit]
Description=Start x11vnc at startup.
After=multi-user.target
[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -xkb -auth guess -forever -loop -noxdamage -repeat -rfbauth /etc/x11vnc.pass -rfbport 5900 -nevershared
[Install]
WantedBy=multi-user.target

# далее включаем x11vnc
sudo systemctl daemon-reload
sudo systemctl enable x11vnc.service
sudo systemctl start x11vnc
```

Подключаться по VNC можно при помощи [mRemoteNG](https://mremoteng.org/download) версии **1.77.1.27713**. С предыдущей версией *mRemoteNG* не работает буфер обмена.

RDP работает лучше VNC, но xrdp создает новую сессию при подключении и не может подключиться к сессии, запущенной автологином.
Данный вопрос решается, если использовать xrdp в сочетании с x11vnc, как прокси. Но в этом случае не работает перенаправление буфера обмена.
Починка буфера обмена для связки xrdp c x11vnc представлена [здесь](https://c-nergy.be/blog/?p=9285&cpage=1)

Обычная установка RDP очень простая:

```
$ yes | sudo apt-get install xrdp
$ sudo systemctl enable xrdp
$ sudo systemctl start xrdp
```

Чтобы xrdp работал как прокси и подключал к сессии x11vnc, нужно прописать в файле *xrdp.ini* подключение к порту x11vnc:

```
sudo nano /etc/xrdp/xrdp.ini
# редактируем настройки. закомментируем секции с настройками для Xorg и т.д. 
# оставляем только одну сессию xrdp0
# также изменяем 

[xrdp0]
name=default
lib=libvnc.so
username=ask
password=ask
ip=127.0.0.1
port=5900
```
Также надо изменить в файле *sesman.ini* параметр X11DisplayOffset на 0:

```
sudo nano /etc/xrdp/sesman.ini

# редактируем параметр X11DisplayOffset

[Sessions]
;; X11DisplayOffset - x11 display number offset
; Type: integer
; Default: 10
X11DisplayOffset=0
```

После чего нужно перезапустить xrdp:

```
sudo systemctl restart xrdp
```

Если при подключении к xrdp показывает [черный экран](https://itobereg.ru/linux/install-xrdp-ubuntu20-04?ysclid=ll7xntbiox532784293#install-gnome-XFCE), необходимо зайти в папку /etc/xrdp, и внести изменения в файл startwm.sh:

```
sudo nano /etc/xrdp/startwm.sh

# Перед строкой test –x /etc/X11/Xsession && exec /etc/X11/Xsession нужно добавить:

unset DBUS_SESSION_BUS_ADDRESS
unset XDG_RUNTIME_DIR

# После внесения изменений необходимо перезапустить службу XRDP

sudo systemctl restart xrdp
```

[Информация про параметры xrdp.ini](https://man.fyi/f26/5+xrdp.ini)

### 6. Установка и настройка автологина сессии

Автологин нужен, чтобы при старте сервера начиналась сессия и без подключения к нему по VNC/RDP. 
В свою очередь сессия нужна, чтобы работала автозагрузка программ.

Для этих целей можно использовать [LightDM](https://wiki.archlinux.org/title/LightDM)

Установка:

```
sudo apt install lightdm lightdm-gtk-greeter
```

Возможно еще придется установить недостающие пакеты:

```
# Для устранения ошибок типа PAM unable to dlopen(/usr/lib/security/pam_gnome_keyring.so)
sudo apt install gnome-keyring

# Для устранения ошибок типа PAM unable to dlopen(pam_kwallet.so): /lib/security/pam_kwallet.so: cannot open shared object file: No such file or director
sudo apt-cache search kwallet |grep pam

# libpam-kwallet4 - KWallet (KDE 4) integration with PAM
# libpam-kwallet5 - KWallet (Kf5) integration with PAM

sudo apt-get install libpam-kwallet4 libpam-kwallet5
sudo service lightdm restart
```

Далее нужно отредактировать файл:

```
sudo nano /etc/lightdm/lightdm.conf
```

Необходимо прописать в нем следующее

```
[Seat:*]
autologin-user=имя_пользователя
```

Также нужно добавить пользователя в группу:

```
sudo groupadd -r autologin
sudo gpasswd -a имя_пользователя autologin
```

И можо сделать включение интерактивного входа без пароля:

```
sudo groupadd -r nopasswdlogin
sudo gpasswd -a имя_пользователя nopasswdlogin
```

Далее нужно сделать перезагрузку. После чего сервер должен сам заходить в сессию после старта.

Для просмотра логов LightDM:

```
sudo journalctl -u lightdm
```

## Подключение к рабочему столку. Запускаем RDP-клиент

Для Windows хороший вариант RDP-клиента можно скачать здесь: [mRemoteNG](https://mremoteng.org/). Рекомендуется использовать версию **1.77.1.27713**, так как у предыдущей версии не рабоатте буфер обмена с VNC.

Менее привлекательный вариант: [https://www.microsoft.com/en-us/download/details.aspx?id=50042](https://www.microsoft.com/en-us/download/details.aspx?id=50042)

Можно так же использовать стандартную программу "Подключение к удаленному рабочему столу".

* Для подключения откройте приложение Windows Подключение к удаленному рабочему столу. Введите IP-адрес сервера и имя пользователя и нажмите Подключить
* Если возникнет необходимость передавать файлы на сервер, можно указать это в настройках программы для подключения. 
* При подключении появится предупреждение безопасности, это связано с тем, что происходит соединение с ОС семейства Linux. Нажмите *Да*.
* В открывшемся окне в качестве сессии выборе Xorg, введите пароль для пользователя, нажмите *OK*.
* Для Windows можно скачать RDP-клиент: https://www.microsoft.com/en-us/download/details.aspx?id=50042

## Установка Wine

Для работы Metatrader на Linux нужна такая полезная штука, как Wine. Wine позволяет запускать приложения Windows на Linux.

#### Проблемы, связанные с Wine

* Программы, использующие подключение к интернету, могут иметь проблемы. Эти проблемы могут быть следствием слишком длинного имени сервера / комьютера. Имя компьютера доджно быть не длинее 15-ти символов.

Имя компьютера можно поменять в следующих файлах:

```
notepad /etc/hostname
notepad /etc/hosts
```

Пример имени сервера / компьютера в файлах.

hostname:

```
my-pc
```

hosts:

```
127.0.0.1	localhost
127.0.1.1	ubuntu
127.0.0.1	my-pc

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

* Во время установки может возникнуть проблема *apt-add-repository command not found*, ее можно решить так:

```
sudo apt install software-properties-common
```

Иногда система может выдавать, что пакет установлен, но несмотря на это продолжать сыпать ошибки при попытке установить PPA. Такое случается иногда из-за ошибок во время установки. Система думает, что пакет установлен, но на самом деле, в файловой системе нет файлов данного пакета, для решения проблемы мы можем его переустановить:

```
sudo apt install --reinstall software-properties-common
```

Чтобы убедиться что пакет установлен правильно и все файлы есть там, где они и должны быть, вы можете использовать команду:

```
dpkg -L software-properties-common
```

Затем можете попытаться выполнить файл программы напрямую:

```
sudo /usr/bin/add-apt-repository
```

* Проблемы с отображением русского языка

Проблема решается при помощи установки русской локали
https://linux-notes.org/nastrojka-locate-lokali-v-unix-linux/


```
localedef -i ru_RU -f UTF-8 ru_RU.UTF-8
```

Для проверки

```
locale -a | grep ru
```

Еще по теме:

https://linux-user.ru/distributivy-linux/programmy-dlya-linux/wine/problemy-s-kodirovkoj-v-programmah-pod-wine/

#### Команды для установки Wine

* Проще установить старую версию Wine, ее вполне достаточно для работы.

```
sudo dpkg --add-architecture i386
wget -nc https://dl.winehq.org/wine-builds/Release.key
sudo apt-key add Release.key
sudo apt install software-properties-common
sudo add-apt-repository "deb https://dl.winehq.org/wine-builds/ubuntu/ artful main"
sudo apt-get update
yes | sudo apt-get install --install-recommends winehq-stable
```

* Если возникли проблемы с установкой МТ4 (просит указать прокси), то можно переустановить wine. Вот две инструкции: 
https://forum.ubuntu.ru/index.php?topic=230947.0
https://www.maksiv.ru/kak-polnostyu-udalit-wine-iz-sistemy/

Команды из инструкций:

```
sudo apt-get remove winetricks
sudo apt autoremove
wine --version
```

Узнаем версию, далее вставляем ее БЕЗ ПРОБЕЛА

```
sudo apt-get remove wineX
```

И выполняем команду

```
sudo apt autoremove
```

Далее можно установить wine повторно:

```
sudo dpkg --add-architecture i386
sudo add-apt-repository ppa:wine/wine-builds
sudo apt-get update
sudo apt-get install --install-recommends winehq-devel
```

При помощи Wine можно запускать **.exe** и даже **.bat** файлы. 
Пример c **.exe**:

```
wine /root/_repoz/example/bin/example.exe
```

Пример c **.bat**:

```
wine start example.bat
```

Также можно задать директорию выполнения 

```
wine start /d "/root/_repoz/" example.bat
```

Про Linux скрипты можно почитать здесь: https://linuxrussia.com/sh-ubuntu.html

### Создание лаунчера для запуска программы через Wine

Создаем лаунчер, прописываем имя и настройки:

```
# пример настроек
# в примере программа находится в папке /home/example

Name:
Test

Command: 			
env WINEPREFIX="/home/имя_пользователя/.wine" wine start /unix /home/example/example.exe

Working Directory: 	
/home/example
```

Также отмечаем галочку напротив опции:

```
Use startup notification
```

Сохраняем настройки лаунчера. Затем делаем файл исполняемым:

```
chmod +x имя_файла.desktop
```

имя_файла.desktop - это наш лаунчер.

## Автозагрузка приложения

Если у нас есть лаунчер, его можно добавить в автозагрузку. Для этого нужно зайти в настройки:

```
Applications->Settingis->Session and Startup->Application Autostart
```

Добавляем наш лаунчер в автозагрузку.

## Установка Google Chrome или Chromium

Отдать приоритет установке **Google Chrome**. Если не выходит установить данный браузер, перейти на **Chromium**.

**Google Chrome**

Оригианл инструкции по установке Google Chrome: https://losst.ru/ustanovka-chrome-v-ubuntu-18-04
Установка стабильной версии Chrome в Ubuntu 64 бит:

```
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i --force-depends google-chrome-stable_current_amd64.deb
```

Запуск браузера

```
google-chrome --no-sandbox
```

**Установка Cromium**

Отличия Chrome от Cromium незначительные:
* Отсутствует поддержка таких закрытых кодеков, как: AAC, H.264 и MP3;
* Отсутствует Adobe Flash;
* Отсутствует механизм обновления для операционных систем MacOS и Windows;
* Отсутствует механизм сбора и передачи некоторых данных о работе браузера на сервера Google;

```
yes | sudo apt install chromium-browser
```
## Установка zip архиватора

```
sudo apt-get install unzip
```

или

```
sudo apt install unzip
```

Архивирование папки:

```
 zip archive.zip -r /path/to/files/*
```

## Настройка времени сервера

Если [сбивается время на сервере](https://losst.ru/sbivaetsya-vremya-v-ubuntu-i-windows), необходимо выполнить команду:

```
sudo timedatectl set-local-rtc 1 --adjust-system-clock
```

Проверить время можно командой:

```
sudo timedatectl
```

[Настроить сбившееся время можно так](https://itshaman.ru/articles/257/sinkhronizatsiya-vremeni-cherez-internet-v-ubuntu):

1) Установить NTP

```
sudo apt-get install ntp
```

2) Открыть файл настроек

```
sudo notepad /etc/ntp.conf
```

И добавить туда сервера

```
ntp1.imvp.ru
ntp.psn.ru
time.nist.gov
pool.ntp.org
ru.pool.ntp.org
```

3) Настраиваем автоматическую синхронизацию при каждой загрузке ОС. Для этого открываем конфигурационный файл /etc/rc.conf:

```
sudo notepad /etc/rc.conf
```

Редактируем параметр ntpd_enable

```
ntpd_enable=»YES»
```

4) После каждого включения компьютера ваше время будет синхронизировано через Интернет и всегда будет актуальным. Если есть необходимость синхронизировать время вручную, то делается это командой:

```
// устаревшая версия
sudo ntpdate time.nist.gov
// новая версия
sntp -sS time.nist.gov
```

В остальном - есть множество статей на данную тему:

https://linuxconfig.org/how-to-change-timezone-on-ubuntu-18-04-bionic-beaver-linux
https://askubuntu.com/questions/323131/setting-timezone-from-terminal/524362#524362
https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-18-04/
https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt
https://pingvinus.ru/note/timezone#tzChangeLocal
https://losst.ru/sbivaetsya-vremya-v-ubuntu-i-windows

Например, можно сделать так:

```
sudo dpkg-reconfigure tzdata
```

Далее, чтбы выбрать время UTС, необходимо выбрать в первом окне *None of the above* или *Etc*, во втором окне *UTC*

Еще вариант:

```
timedatectl set-timezone UTC
```

## Установка 3proxy

Данный шаг может пригодиться, если нужно пройти верификацию аккаунта через тот же самый IP адрес. 
Но в случае с покупкой IP адреса и использованием VPN на телефоне - в этом нет смысла. Поэтому данный шаг пропускается за редким исключением.

Оригинал статьи: [https://www.dmosk.ru/miniinstruktions.php?mini=3proxy-ubuntu](https://www.dmosk.ru/miniinstruktions.php?mini=3proxy-ubuntu)
Статьи по теме: 
[http://prosoft-m.ru/225-kak-sozdat-proksi-server.html](http://prosoft-m.ru/225-kak-sozdat-proksi-server.html)
[https://sysadmin.pm/3proxy/](https://sysadmin.pm/3proxy/)
[https://www.dmosk.ru/miniinstruktions.php?mini=3proxy-centos}(https://www.dmosk.ru/miniinstruktions.php?mini=3proxy-centos)


### Шаг первый. Настройка брандмауэра

```
iptables -I INPUT 1 -p tcp --dport 3128 -j ACCEPT
netfilter-persistent save
```

Или

```
firewall-cmd --permanent --add-port=3128/tcp
firewall-cmd --reload
```

3128 — порт по умолчанию, по которому работает 3proxy в режиме прокси;

### Шаг второй. Установка и запуск 3proxy

Устанавливаем пакет программ для компиляции пакетов:

```
sudo apt-get install build-essential
```

Официальная страница загрузки 3proxy [https://3proxy.org/download/stable/](https://3proxy.org/download/stable/) и [http://prosoft-m.ru/225-kak-sozdat-proksi-server.html](http://prosoft-m.ru/225-kak-sozdat-proksi-server.html)
Используя ссылку, скачиваем пакет

```
wget https://github.com/z3APA3A/3proxy/archive/0.8.12.tar.gz
```

Распакуем скачанный архив:

```
tar xzf 0.8.12.tar.gz
```

Переходим в распакованный каталог:

```
cd 3proxy-*
```

Запускаем компиляцию *3proxy*:

```
make -f Makefile.Linux
```

Создаем системную учетную запись:

```
sudo adduser --system --disabled-login --no-create-home --group proxy3
```

* Примечание, *adduser* без *sudo* не работает.

Создаем каталоги и копируем файл *3proxy* в */usr/bin*:

```
mkdir -p /var/log/3proxy
mkdir /etc/3proxy
cp src/3proxy /usr/bin/
```

Задаем права на созданные каталоги:

```
chown proxy3:proxy3 -R /etc/3proxy
chown proxy3:proxy3 /usr/bin/3proxy
chown proxy3:proxy3 /var/log/3proxy
```

Смотрим uid и gid созданной учетной записи:

```
id proxy3
```

Получим, примерно, такой результат:

```
uid=109(proxy3) gid=113(proxy3) groups=113(proxy3)
```

Создаем конфигурационный файл:

```
nano /etc/3proxy/3proxy.cfg
```

```
setgid 121
setuid 112

nserver 77.88.8.8
nserver 8.8.8.8

nscache 65536

timeouts 1 5 30 60 180 1800 15 60

users user2020:CL:ab34gghy

external 185.253.218.12 
internal 185.253.218.12

daemon

log /var/log/3proxy/3proxy.log D
logformat "- +_L%t.%. %N.%p %E %U %C:%c %R:%r %O %I %h %T"

auth strong

allow * * * 80-88,8080-8088 HTTP
allow * * * 443,8443 HTTPS

socks -p8083 -i185.253.218.12 -e185.253.218.12
proxy -n
```

* необходимо обратить внимание на настройки setgid и setuid — это должны быть значения для созданной нами учетной записи; external и internal — внешний и внутренний интерфейсы (если наш прокси работает на одном адресе, то IP-адреса должны совпадать). 
* возможные типы паролей:
CL — текстовый пароль
CR — зашифрованный пароль (md5)
NT — пароль в формате NT.
* Имя и пароль - *user2020* и *ab34gghy*
* Чтобы включить запрос логина, необходимо поменять значение для опции auth на strong (уже настроено в примере, без пароля будет auth none)
* запускаем socks на порту 8083; внутренний интерфейс — 192.168.1.23, внешний — 111.111.111.111.

Запускаем 3proxy:

```
/usr/bin/3proxy /etc/3proxy/3proxy.cfg
```

В браузере можно установить расширение [FoxyProxy](https://chrome.google.com/webstore/detail/foxyproxy-standard/gcknhkkoolaabfmlnjonogaaifnjlfnp/related?hl=ru) и через него подключиться к прокси.

## Установка Metatrader

Metatrader устанавливается как обычно. Есть лишь некоторые ньюансы:

* Папка песочницы Metatrader находится здесь: *home\.wine\drive_c\Program Files(x84)\MetaTrader 4*
* Терминал может глючить, подвисают котировки (https://www.mql5.com/ru/forum/116975) или слетают настройки (https://www.mql5.com/ru/forum/281434). Поэтому рекомендуется сохранять шаблоны и профили http://www.tevola.ru/trading/torgovye-platformy/metatrader/profiles-i-template-v-metatrader.html
* При работе с FXCM для потока котировок лучше использовать сервер FXCMUSD-Demo02 Австралия.

## Полезные ссылки:

[VPS на Linux с графическим интерфейсом: запускаем сервер RDP на Ubuntu 18.04](https://habr.com/ru/companies/ruvds/articles/512878/#section5)
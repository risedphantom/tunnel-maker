# tunnel-maker

Скрипт-обертка над ssh, который позволяет "в один клик" пробрасывать тунели до баз и сервисов на выбранном сервере. Для нормального функционирования необходимо добавить аутентификацию по ключу на данном сервере. 

***Установка:***

Находясь в папке проекта:
```
sudo apt install python3-pip
pip3 install psutil
mkdir ~/bin
echo 'PATH=$HOME/bin:$PATH' >> ~/.profile
. ~/.profile
cp tunnel ~/bin
```

***Основные фичи:***
```
usage: tunnel [-h] [-e ENV] [-u USER] [-s] [-t] [-k] [-r] [-l]
              [-p PORTS [PORTS ...]]

-= Tunnel maker =-

optional arguments:
  -h, --help            show this help message and exit
  -e ENV, --env ENV     specify environment for tunnels (default: None)
  -u USER, --user USER  specify username for tunnels (default: None)
  -s, --show            show list of active tunnels (default: False)
  -t, --tty             open ssh connection to specified env (default: False)
  -k, --kill            terminate active tunnels (default: False)
  -r, --remote          backdoor remote port to local (default: False)
  -l, --local           map local port to remote (default: False)
  -p PORTS [PORTS ...], --port PORTS [PORTS ...]
                        ports list or known applications aliases (default: [])

Examples:
  tunnel                                      - show all active tunnels
  tunnel -t                                   - open ssh connection to specified env
  tunnel -p redis                             - show active tunnels for redis
  tunnel -e box01 -l -p redis 6003            - forward ports for redis and 6003 port to sandbox-01
  tunnel -r -p 6003                           - backdoor port for api service to default env
  tunnel -k                                   - kill all tunnels
  tunnel -kp api                              - kill tunnels for api service

```

***Как вообще этим пользоваться?***

Допустим иы хотим пробросить 2 тунеля до баз данных - MongoDB и Redis:
```
tunnel -lp mongo redis
```
Будут созданы 2 ssh тунеля. Посмотреть открытые тунели можно простой командой
```
tunnel
```
Увидим:
```
 PID  NAME             LPORT  DESTINATION   RPORT  ENV
4021  Redis             6379  --FORWARD-->   6379  box01
4023  MongoDB          27017  --FORWARD-->  27017  box01
```

***Теперь чуть посложнее...***

Мы хотим локально дебажить сервис example-service (порт 6003), который использует монгу и редис.
При этом запросы мы хотим отправлять на сэндбокс. Допустим на сэндбоксе нет монги и она видна по адресу mongodb-some-host.com, но только внутри сети.

Для этого мы заходим на сэндбокс. Убиваем службу example-service
```
sudo systemctl stop example-service
```
Пробрасываем тунели
```
tunnel -lp redis mongo
```
Пробрасываем обратный бэкдор для example-service
```
tunnel -rp 6003
```
Смотрим список тунелей
```
tunnel
```
И видим следующее:
```
 PID  NAME             LPORT  DESTINATION   RPORT  ENV
4293  Mongo            27017  ---PROXY--->  27017  box01
4294  Redis             6379  --FORWARD-->   6379  box01
4300  UNKNOWN           6003  <----BACK---   6003  box01
```

Local port forwarding обозначается как --FORWARD--> (прямой проброс портов)
Remote port forwarding видим как <----BACK--- (обратный проброс)
Проксирование до монги выглядит как ---PROXY--->

Дефолтный сэндбокс, на который ходить, прописывается ручками в самом начале скрипта, если лень каждый раз указывать параметром
Работает на 3 питоне (есть из коробки в убунте), однако потребудется установить пакет psutil

# tunnel-maker

Скрипт-обертка над ssh, которая позволяет "в один клик" пробрасывать тунели до баз и сервисов на выбранном сервере. Для нормального функционирования необходимо добавить аутентификацию по ключу на данном сервере. 

***Установка:***

Находясь в папке проекта:
```
sudo apt install python-pip
pip install psutil
mkdir ~/bin
echo 'PATH=$HOME/bin:$PATH' >> ~/.profile
. ~/.profile
cp tunnel ~/bin
```

***Основные фичи:***
```
usage: tunnel [-h] [-e ENV] [-s] [-k] [-r] [-l] [-p PORTS [PORTS ...]]

-= Tunnel maker =-

optional arguments:
  -h, --help            show this help message and exit
  -e ENV, --env ENV     specify environment for tunnels (default: None)
  -s, --show            show list of active tunnels (default: False)
  -k, --kill            terminate active tunnels (default: False)
  -r, --remote          backdoor remote port to local (default: False)
  -l, --local           map local port to remote (default: False)
  -p PORTS [PORTS ...], --port PORTS [PORTS ...]
                        ports list or known applications aliases (default: [])

Examples:
  tunnel                                      - show all active tunnels
  tunnel -p avia                              - show active tunnels for avia
  tunnel -e sandbox-23 -lp avia 8881        - forward ports for avia and stats to sandbox-23
  tunnel -r -p 8881                           - backdoor port for stats to default env
  tunnel -k                                   - kill all tunnels
  tunnel -kp stats                            - kill tunnels for stats
```

***Как вообще этим пользоваться?***

Допустим иы хотим пробросить 2 тунеля до баз данных - Riak и Redis:
```
tunnel -lp riak redis
```
Будут созданы 2 ssh тунеля. Посмотреть открытые тунели можно простой командой
```
tunnel
```
Увидим:
```
 PID  NAME             LPORT  DESTINATION   RPORT  ENV
4021  Redis             6379  --FORWARD-->   6379  sandbox-23
4023  Riak              8098  --FORWARD-->   8098  sandbox-23
```

***Теперь чуть посложнее...***

Мы хотим локально дебажить сервис example-service (порт 8012), который использует монгу, риак и редис.
При этом запросы мы хотим отправлять на сэндбокс. Допустим на сэндбоксе нет монги и она видна по адресу mongodb-some-host.com, но только внутри сети.

Для этого мы заходим на сэндбокс. Убиваем службу example-service
```
sudo systemctl stop example-service
```
Пробрасываем тунели
```
tunnel -lp redis riak mongo
```
Пробрасываем обратный бэкдор для example-service
```
tunnel -rp 8012
```
Смотрим список тунелей
```
tunnel
```
И видим следующее:
```
 PID  NAME             LPORT  DESTINATION   RPORT  ENV
4289  Riak              8098  --FORWARD-->   8098  sandbox-23
4293  Mongo            27017  ---PROXY--->  27017  sandbox-23
4294  Redis             6379  --FORWARD-->   6379  sandbox-23
4300  UNKNOWN           8012  <----BACK---   8012  sandbox-23
```

Local port forwarding обозначается как --FORWARD--> (прямой проброс портов)
Remote port forwarding видим как <----BACK--- (обратный проброс)
Проксирование до монги выглядит как ---PROXY--->

Дефолтный сэндбокс, на который ходить, прописывается ручками в самом начале скрипта, если лень каждый раз указывать параметром
Работает на 3 питоне (есть из коробки в убунте), однако потребудется установить пакет psutil

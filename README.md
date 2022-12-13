
# OSPF
---
## Поднимаем три виртуалки и асинхронный OSPF между ними.

- Для этого создаем ansible-playbook, в котором установим полезны утилиты тип tcpdump и traceroute.
- Остановим файрволл, добавим gpg и репозиторий frr.
- Включим forwarding и выключим проверку входящих пакетов на проверку того, что ip-пакет приходит на тот интерфейс с которго был отправлен. Для того чтобы можно было включить асинхронный роутинг.
- Создадим два файла темплейта в папке templates- daemons.j2 и frr.conf.j2. В файле frr.conf.j2 опишем разницу в настройках ospf для разных роутеров в файле frr.conf.j2 c помощью ansible_facts.
- В папке defaults создадим файл main.yml, в который объявим переменную symmetric_routing: false - для того чтобы можно было включить синхронный режим роутинга в дальнейшем.
- Добавим в плейбук таск на копирование темплейтов и перезапуск с тэгами  - setup_routing для удобства смены режимов роутинга. 
- По умолчанию стенд запустится в асинхронном режиме роутинга и для интерфейса router3, который смотрит в сторону router1 будет выставлен ospf cost 1000. Это значит, что пинги в сеть 192.168.10.0 от router3 будут идти через router2 а ответы будут приходить с другого интерфейса. Проверим это. 
---
Запустим стенд командой vagrant up && ansible-playbook ospf.yml и зайдем на router3, убедимся что на нем есть маршруты во все сети и они доступны.
```console
root@router3:~# vtysh

Hello, this is FRRouting (version 8.4.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router3# sh ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

K>* 0.0.0.0/0 [0/100] via 10.0.2.2, enp0s3, src 10.0.2.15, 00:10:27
C>* 10.0.2.0/24 is directly connected, enp0s3, 00:10:27
K>* 10.0.2.2/32 [0/100] is directly connected, enp0s3, 00:10:27
O>* 10.0.10.0/30 [110/200] via 10.0.11.2, enp0s8, weight 1, 00:09:42
O   10.0.11.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:10:26
C>* 10.0.11.0/30 is directly connected, enp0s8, 00:10:27
O   10.0.12.0/30 [110/300] via 10.0.11.2, enp0s8, weight 1, 00:09:42
C>* 10.0.12.0/30 is directly connected, enp0s9, 00:10:27
O>* 192.168.10.0/24 [110/300] via 10.0.11.2, enp0s8, weight 1, 00:09:42
O>* 192.168.20.0/24 [110/200] via 10.0.11.2, enp0s8, weight 1, 00:09:42
O   192.168.30.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:10:26
C>* 192.168.30.0/24 is directly connected, enp0s10, 00:10:27
C>* 192.168.56.0/24 is directly connected, enp0s16, 00:10:27

router3# traceroute 192.168.10.1
traceroute to 192.168.10.1 (192.168.10.1), 30 hops max, 60 byte packets
 1  10.0.11.2 (10.0.11.2)  1.067 ms  0.947 ms  0.869 ms
 2  192.168.10.1 (192.168.10.1)  1.722 ms  1.351 ms  1.297 ms

root@router3:~# tcpdump -i enp0s9 icmp
09:38:40.739340 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 1, length 64
09:38:41.740979 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 2, length 64
09:38:42.828213 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 3, length 64
09:38:43.866126 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 4, length 64
09:38:44.868293 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 5, length 64
09:38:45.871321 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 6, length 64
09:38:46.872558 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 7, length 64
09:38:47.875126 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 8, length 64
09:38:48.876424 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 9, length 64
09:38:50.006451 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 10, length 64
09:38:51.008666 IP 192.168.10.1 > router3: ICMP echo reply, id 4, seq 11, length 64

root@router3:~# tcpdump -i enp0s8 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
09:38:58.060913 IP router3 > 192.168.10.1: ICMP echo request, id 4, seq 18, length 64
09:38:59.062884 IP router3 > 192.168.10.1: ICMP echo request, id 4, seq 19, length 64
09:39:00.064963 IP router3 > 192.168.10.1: ICMP echo request, id 4, seq 20, length 64
09:39:01.066510 IP router3 > 192.168.10.1: ICMP echo request, id 4, seq 21, length 64
09:39:02.078308 IP router3 > 192.168.10.1: ICMP echo request, id 4, seq 22, length 64
```
Видим, что все маршруты есть, icmp request уходит с интерфейса enp0s8, а ответы приходят с enp0s9 - асинхронный роутинг работает.

---
- Для того, чтобы сделать роутинг симметричным изменим переменную symmetric_routing в файле defaults/main.xml на true. 
- Запустим команду ansible-playbook ospf.yml -t "setup_routing". 
- Команда изменит ospf cost на интерфейсе router2, который смотрит в сторону router1 на 1000, что приведет к тому что трафик с router3 пойдет по прямому интерфейсу между router1 и router3 - таким образом роутинг станет опять симметричным.
---
Зайдем на router3 и проверим что пинги уходят и приходят на один и тот же интерфейс.
```console
root@router3:~# tcpdump -i enp0s9 icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
09:57:00.415517 IP router3 > 192.168.10.1: ICMP echo request, id 5, seq 7, length 64
09:57:00.416500 IP 192.168.10.1 > router3: ICMP echo reply, id 5, seq 7, length 64
09:57:01.417377 IP router3 > 192.168.10.1: ICMP echo request, id 5, seq 8, length 64
09:57:01.418401 IP 192.168.10.1 > router3: ICMP echo reply, id 5, seq 8, length 64
09:57:02.419264 IP router3 > 192.168.10.1: ICMP echo request, id 5, seq 9, length 64
09:57:02.420214 IP 192.168.10.1 > router3: ICMP echo reply, id 5, seq 9, length 64
09:57:03.420923 IP router3 > 192.168.10.1: ICMP echo request, id 5, seq 10, length 64
```
Вывод: роутинг симметричный т.к. пинги уходят и приходят на один и тот же интерфейс. Задание выполнено.

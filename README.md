# iptables-
Фильтрация трафика - iptables 
1. Для выполнения данного ДЗ берем схему из ДЗ по "Архитектуре сетей"  https://github.com/novohudanosor/LAN
2. port knocking — автоматизированный метод внешнего открытия TCP- и UDP-портов в брандмауэре отправкой определённых пакетов на заранее указанные закрытые порты. Когда будет получена верная последовательность, правила брандмауэра автоматически изменятся для разрешения клиенту, который её отправил, подключаться к ранее защищённым портам. Основной целью этого метода является защита потенциально уязвимого программного обеспечения от брутфорса и прочих атак, ибо если атакующий не знает верную последовательность, то защищённые порты будут казаться закрытыми, тем самым создавая вид, что на них ничего не работает. Зачастую этот метод используется для защиты SSH. (Взято из Википедии)
3. Пробуем реализовать port knocking с помощью iptables.
* Создаем цепочки TRAFFIC, SSH-INPUT, SSH-INPUTTWO:
```
iptables -N TRAFFIC
iptables -N SSH-INPUT
iptables -N SSH-INPUTTWO
```
* правила для цепочки TRAFFIC
```
iptables -A INPUT -i enp0s8 -j TRAFFIC
iptables -A TRAFFIC -i enp0s8 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT
```
* В примере выше происходит открытие 22-го порта на 30 секунд, при условии, что подключающийся ip находится в списке SSH2. Если в соответствии с данным правилом трафик не принимается (например, не было попыток подключения в течение 30 секунд), но подключающийся IP-адрес находится в списке SSH2, он удаляется из этого списка для осуществления проверки с самого начала.
```
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
```
* После того, как конец последовательности был обработан первым, следующие правила выполняют проверку следования портов. Если проверка пройдена, то происходит переход к цепочке, в которой ip-адрес добавляется в список для следующего запроса. Если перехода на цепочки SSH-INPUT или SSH-INPUTTWO не произошло, то это может свидетельствовать о неверных портах в последовательности или о наличии неустановленного трафика. После чего, по аналогии с предыдущим правилом для списка SSH2, осуществляется удаление ip-адреса из списка SSH1 и отбрасывание трафика.
```
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
```
* Так же делаем для следующего порта.  Порядок следования в цепочке TRAFFIC может быть любым при условии, что правила, соответствующие одному и тому же списку, сохраняются вместе и в правильном порядке.
```
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
``` 
* В этой цепочке, осуществляется попытка установления соединения для ip, соответствующего списку разрешённых ip-адресов. Первый порт в последовательности проверяется как часть основной цепочки TRAFFIC, поскольку каждая новая попытка подключения может являться началом процедуры port knocking. В случае успеха проверки (правильный порт), порт записывается в первый список SSH0. Далее в правиле для проверки второго порта (7777), из списка SSH0 требуется подтверждение проверки первого порта, только после этого производится запись о втором порте в список SSH1. В свою очередь, список SSH1 требуется для проверки третьего порта (9991). В последних правилах блока осуществляется сброс трафика для портов из списков, несмотря на то, что они корректны. Эти действия необходимы для сокрытия успешности подключения к каждому из портов.
```
iptables -A TRAFFIC -i enp0s8 -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP
iptables -A SSH-INPUT -m recent --name SSH1 --set -j DROP
iptables -A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP
iptables -A TRAFFIC -i enp0s8 -j DROP
```
*  Что бы постучаться со стороны клиента, используем nmap. С помощью скрипта
```
#!/bin/bash
HOST=$1
shift

for ARG in "$@"
do
   nmap -Pn -e enp0s8 --max-retries 0 -p $ARG $HOST
done
``` 
*  Проверка port knocking
```
vagrant@centralRouter:~$ ssh vagrant@192.168.255.1
^C

vagrant@centralRouter:~$ ./knock 192.168.255.1 8881 7777 9991
Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-20 13:04 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up.

PORT     STATE    SERVICE
8881/tcp filtered galaxy4d

Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds
Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-20 13:04 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up.

PORT     STATE    SERVICE
7777/tcp filtered cbt

Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds
Starting Nmap 7.80 ( https://nmap.org ) at 2024-07-20 13:04 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up.

PORT     STATE    SERVICE
9991/tcp filtered issa

Nmap done: 1 IP address (1 host up) scanned in 14.02 seconds

vagrant@centralRouter:~$ ssh vagrant@192.168.255.1
The authenticity of host '192.168.255.1 (192.168.255.1)' can't be established.
ED25519 key fingerprint is SHA256:dEof/6Fm9Iy3IvzJwcYS1k0EAtuEJAVNkVA7dZbaPBU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.255.1' (ED25519) to the list of known hosts.
vagrant@192.168.255.1's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-97-generic x86_64)
...

``` 
## Проброс портов 
* После установки NGINX на centralServer (192.168.0.2), необходимо пробросить порт 8080 inetRouter2 (192.168.255.14) на порт 80 centralServer. Данное действие осуществляется с помощью добавления правила в iptables:
```
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to 192.168.0.2:80
``` 
## пробросить 80й порт на inetRouter2 8080
```
iptables -t nat -A POSTROUTING -o enp0s8 -p tcp -d 192.168.0.2 --dport 80 -j SNAT --to-source 192.168.255.14:8080
``` 




### Название  
__ferm__ - парсер файерволл-правил для линукс  

### Синтаксис  
__ferm__ _options inputfile_  

### Описание  
__ferm__ это фронтенд для __iptables__. Он читает правила из структурированного конфигурационного файла и вызывает iptables(8) для вставки их в запущенное ядро.  

Задача __ferm__'а сделать файерволл-правила более простыми для написания и чтения. Он пытается сократить утомительную задачу записи правил, так что это позволяет файерволл-администратору тратить больше времени на разработку хороших правил чем на надлежащую имплементацию обычных правил.  

Для достижения этого, __ferm__ использует простой, но мощный язык конфигурирования, в котором доступны переменные, функции, массивы и блоки. Он также позволяет тебе инклудить другие файлы, разрешая этим создавать библиотеки часто используемых структур и функций.  

__ferm__ произносится как "firm", и означает "For Easy Rule Making".  

### Предупреждение  
Этот мануал _не_ стремится научить тебя тому как работает файерволл и как писать хорошие правила. На эту тему уже есть достаточно документации.  

### Вступление  
Начнем с простого примера:  
```
chain INPUT {
  proto tcp ACCEPT;
}
```
Здесь добавляется правило во встроенную цепочку INPUT, которое сопоставляет и принимает все TCP пакеты. Хорошо, давайте усложним его:  
```
chain (INPUT OUTPUT) {
        proto (udp tcp) ACCEPT;
}
```
Здесь будет вставлено 4 правила, а именно 2 правила в цепочку INPUT и 2 в OUTPUT. Правила будут принимать UDP и TCP пакеты.  
Обычно ты бы напечатал так:  
```
iptables -A INPUT -p tcp -j ACCEPT
iptables -A OUTPUT -p tcp -j ACCEPT
iptables -A INPUT -p udp -j ACCEPT
iptables -A OUTPUT -p udp -j ACCEPT
```
Заметил насколько меньше нам нужно печатать чтобы сделать это? :-)  

В основном это все что нужно делать, однако, ты можешь сделать его более сложным. Что-то похожее на:  
```
chain INPUT {
    policy ACCEPT;
    daddr 10.0.0.0/8 proto tcp dport ! ftp jump mychain sport :1023 TOS 4 settos 8 mark 2;
    daddr 10.0.0.0/8 proto tcp dport ftp REJECT;
}
```
Моя мысль здесь в том, что тебе нужно писать хорошие правила, оставлять их читаемыми для тебя и других и не превращать их в бардак.  

Наличие результирующих правил здесь для справки, помогло бы читателю. Также ты можешь заинклудить вложенную версию для большей читабельности.  

Попробуй использовать комментарии, чтобы показать что ты делаешь:  
```
# this line enables transparent http-proxying for the internal network:
proto tcp if eth0 daddr ! 192.168.0.0/255.255.255.0
    dport http REDIRECT to-ports 3128;
```
Потом сам скажешь спасибо!  
```
chain INPUT {
    policy ACCEPT;
    interface (eth0 ppp0) {
        # deny access to notorious hackers, return here if no match
        # was found to resume normal firewalling
        jump badguys;

        protocol tcp jump fw_tcp;
        protocol udp jump fw_udp;
    }
}
```
Чем больше вложенность, тем лучше это выглядит. Убедись что твоя последовательность верна, ведь ты бы не хотел сделать такое:  
```
chain FORWARD {
    proto ! udp DROP;
    proto tcp dport ftp ACCEPT;
}
```
потому что второе правило никогда не сработает. Лучший путь это сперва определить все что разрешено, а затем запретить все остальное. Смотри на примеры, чтобы было больше "хороших снимков". Многие люди делают что-то похожее на это:  
```
proto tcp {
    dport (
        ssh http ftp
    ) ACCEPT;
    dport 1024:65535 ! syn ACCEPT;
    DROP;
}
```

### Структура конфигурационного файла  
Структура корректного файерволл-файла похожа на упрощенный C-код. Лишь несколько синтаксических символов используются в конфигурационных файлах ferm. Кроме этих специальных символов, ferm использует 'keys' и 'values', думай об этом как об опциях и параметрах или как о переменных и значениях.  

С этими словами ты опишешь свойства своего файерволла. Каждый файерволл состоит из двух вещей: Первое, проверка подходит ли сетевой трафик под конкретные условия, и второе, что делать с этим трафиком.  

Ты можешь описать условия которые будут валидны для kernel interface program которую ты используешь, скорее всего это iptables(8). Например, в iptables, если ты попытаешься сопоставить TCP пакеты, ты бы написал:  
```
iptables --protocol tcp
```
В ferm это превращается в:  
```
protocol tcp;
```
Просто набранное это, не сделает ничего, тебе нужно сказать ferm'у (на самом деле нужно сказать iptables(8) и ядру), что делать с трафиком который совпадает с этим условием:  
```
iptables --protocol tcp -j ACCEPT
```
Или, в переводе __ferm__'a:  
```
protocol tcp ACCEPT;
```
Символ __;__ есть на конце каждого ferm-правила. Ferm игнорирует переносы строк, значит пример выше идентичен следующему:  
```
protocol tcp
  ACCEPT;
```
Далее список специальных символов:  
##### ;  
Этот символ заканчивает правило. Разделяя точкой с запятой, ты можешь писать множество правил в одну строку, хоть это и уменьшает читабельность:  
```
protocol tcp ACCEPT; protocol udp DROP;
```

##### {}  
Символ вложенности определяет 'блок' правил.  

Фигурные скобки содержат некоторое количество вложенных правил. Все совпадения до блока переносятся к ним.  

Закрывающая фигурная скобка завершает набор правил. Тебе не нужно писать ';' после нее потому что это будет пустым правилом.  

Пример:  
```
chain INPUT proto icmp {
    icmp-type echo-request ACCEPT;
    DROP;
}
```
Этот блок показывает два правила внутри блока, они оба будут объединены с чем-либо перед этим блоком и ты получишь два правила:  
```
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -p icmp -j DROP
```
Здесь может быть множество уровней вложенности:  
```
chain INPUT {
    proto icmp {
        icmp-type echo-request ACCEPT;
        DROP;
    }
    daddr 172.16.0.0/12 REJECT;
}
```
Заметь что на правило 'REJECT' не влияет 'proto icmp', однако там нет ';' после закрывающей фигурной скобки. В переводе на iptables:  
```
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT -p icmp -j DROP
iptables -A INPUT -d 172.16.0.0/12 -j REJECT
```

##### $  
Раскрытие переменной. Заменяет '$FOO' на значение переменной. Смотри в секцию VARIABLES для деталей.  

##### &  
Вызов функции. Смотри секцию FUNCTIONS для деталей.  

##### ()  
Символ массива. Используя круглые скобки ты можешь определить 'список' значений которые должны быть применены к ключу слева от него.  

Пример:  
```
protocol ( tcp udp icmp )
```
результатом этого будут три правила:  
```
... -p tcp ...
... -p udp ...
... -p icmp ...
```
Только значения могут буть 'засписочены', поэтому ты не сможешь сделать что-то похожее на это:  
```
proto tcp ( ACCEPT LOG );
```
но ты можешь сделать так:  
```
chain (INPUT OUTPUT FORWARD) proto (icmp udp tcp) DROP;
```
(что приведет к девяти правилам!)  

Значения разделяются пробелами. Массив одновременно лево- и право-ассоциативный, в отличие от блока вложенности, который может быть только лево-ассоциативным.  

##### \#  
Символ комментария. Все что следует за этим символом до конца строки - игнорируется.  

##### \`command\`  
Выполнить команду в shell и вставить вывод процесса. Смотри секцию _backticks_ для деталей.  

##### 'string'  
Закавыченная строка, которая может содержать пробелы, знаки доллара и т.д.  
```
LOG log-prefix ' hey, this is my log prefix!';
```

##### "string"  
Закавыченная строка (см. выше), но вызовы переменных через знак доллара остаются вычислимыми:  
```
DNAT to "$myhost:$myport";
```

#### Ключевые слова  
В предыдущей секции мы уже представили некоторые базовые ключевые слова, такие как "chain", "protocol" и "ACCEPT". Давайте исследуем их природу.  

Здесь три вида ключевых слов:  
- __location__ ключевые слова определяют где будет создано правило. Например: "table", "chain".  
- __match__ ключевые слова сопоставляют все проходящие пакеты на соответствие условию. Текущее правило будет без эффекта если одно (или больше) сопоставление по критерию не пройдет. Пример: "proto", "daddr".  
  За множеством критериев следует параметр: "proto tcp", "daddr 172.16.0.0./12".  
- __target__ ключевые слова указывают что делать с пакетом. Пример: "ACCEPT", "REJECT", "jump".  
  Многие цели описываются большим количеством ключевых слов для описания деталей: "REJECT reject-with icmp-net-unreachable".  

Каждое правило состоит из __location__ и __target__, плюс некоторое количество __matches__:  
```
table filter                  # location
proto tcp dport (http https)  # match
ACCEPT;                       # target
```
Строго говоря, существует четвертый вид ключевых слов __ferm__ (которые управляют внутренним поведением ferm), но они будут объяснены позже.  

#### Параметры  
Множество ключевых слов используют параметры. Они могут быть определены как константы, вызовы переменных или списки (массивы):  
```
proto udp
saddr $TRUSTED_HOSTS;
proto tcp dport (http https ssh);
LOG log-prefix "funky wardriver alert: ";
```
Некоторые из них могут быть инвертированы (списки не могут):  
```
proto !esp;
proto udp dport !domain;
```
Ключевые слова, которые не принимают параметров отрицаются префиксом '!':  
```
proto tcp !syn;
```
Читай iptables(8) чтобы увидеть где может быть использован __!__.  

### Базовые ключевые слова  
#### location keywords  
- __domain \[ip|ip6\]__ - Устанавливает домен. "ip" значит "IPv4" (iptables) а "ip6" для поддержки IPv6, используя "ip6tables". Если ключевое слово не указано, будет использоваться значение из опции __--domain__ или если и она не указана, то будет выбрано "ip", т.е. IPv4.  
- __table \[filter|nat|mangle\]__ - Определяет в какую таблицу netfilter будет вставлено это правило: "filter" (default), "nat" или "mangle".  
- __chain \[chain-name\]__ - Определяет в какую цепочку netfilter (в текущей таблице) это правило будет вставлено. Обычно в зависимости от таблицы предопределенны следующие имена цепочек "INPUT", "OUTPUT", "FORWARD", "PREROUTING", "POSTROUTING". Смотри документацию к netfilter для справки.  

  Если указать здесь имя несуществующей цепочки, то ferm добавит правило в кастомную цепочку м таким именем.  
- __policy \[ACCEPT|DROP|...\]__ - Определяет политику по умолчанию для текущей цепочки (только встроенные значения). Может быть одной из встроенных целей (ACCEPT, DROP, REJECT, ...). Пакет которому не подойдет ни одно правило в цепочке, будет обработан указанной политикой.  

  Чтобы избежать двусмысленности, всегда указывайте политики всех предопределенных цепочек явно.  
- __@subchain \["CHAIN-NAME"\]{...}__ - Работает как обычный блок (без _@subchain_), за исключением того, что ferm перемещает правила внутри фигурных скобок в новую пользовательскую цепочку. Иия для этой цепочки будет выбрано автоматически ferm'ом.  

  В множестве случаев это быстрее чем просто блок, потому что ядро может пропускать большие блоки правил когда предварительное условие ложно. Представь следующий пример:  
  ```
  table filter chain INPUT {
    saddr (1.2.3.4 2.3.4.5 3.4.5.6 4.5.6.7 5.6.7.8) {
      proto tcp dport (http https ssh) ACCEPT;
      proto udp dport domain ACCEPT;
    }
  }
  ```
  Здесь будет сгенерировано 20 правил. Когда приходит пакет который не проходит __saddr__ сопоставление, он тем не менее проверится через все 20 правил. С __@subchain__ эта проверка пройдет единожды, как результат быстрая фильтрация и меньшая нагрузка на CPU:  
  ```
  table filter chain INPUT {
    saddr (1.2.3.4 2.3.4.5 3.4.5.6 4.5.6.7 5.6.7.8) @subchain {
      proto tcp dport (http https ssh) ACCEPT;
      proto udp dport domain ACCEPT;
    }
  }
  ```
  Опционально можно задать имя для подцепочки:  
  ```
  saddr (1.2.3.4 2.3.4.5 3.4.5.6) @subchain "foobar" {
    proto tcp dport (http https ssh) ACCEPT;
    proto udp dport domain ACCEPT;
  }
  ```
  Именем может быть любая закавыченная строковая константа или развернутое ferm выражение, такое как @cat("interface_", $iface) или @substr($var,0,20).  
  
  Ты можешь добиться такого же результата явно описывая кастомные цепочки, но ты можешь почувствовать, что использование __@subchain__ требует меньше ввода.  
- __@gotosubchain \["CHAIN-NAME"\]{...}__ - Работает как __@subchain__ за искллючением того что вместот использования __jump__ target используется __goto__ target. Разницу между этими двумя таргетами смотри ниже.  
- __@preserve__ - Сохранять существующие правила в текущей цепочке:  
  ```
  chain (foo bar) @preserve;
  ```
  С этой опцией __ferm__ загружает предыдущий набор правил используя __iptables-save__,  выделяет все "preserved" цепочки и вставляет их в итоговый вывод.  
  
  "Preserved" цепочки не должны изменяться __ferm__'ом: ни правила, ни политики.  

#### Basic iptables match keywords  
- __interface \[interface-name\]__ - Определяет имя интерфейса твоей внешней сетевой карты, как eth0 или dialup ppp1, или какого-либо другого девайса имя которого ты хочешь сопоставлять с проходящими пакетами. Это эквивалентно `-i` в iptables(8).  
- __outerface \[interface-name\]__ - То же что и interface, только пакеты сопоставляются с исходящим интерфейсом, как в iptables(8).  
- __protocol \[protocol-name|protocol-number\]__ - Сейчас ядро поддерживает TCP, UDP и ICMP или их соответствующие номера.  
  Вместо __protocol__ ты можешь также использовать сокращение __proto__.  
- __saddr|daddr \[address-spec\]__ - Совпадение указанного адреса в адресе отправителя (saddr) или в адресе получателя (daddr).  
  Пример:  
  ```
  saddr 192.168.0.0/24 ACCEPT; # (identical to the next one:)
  saddr 192.168.0.0/255.255.255.0 ACCEPT;
  daddr my.domain.com ACCEPT;
  ```
- __fragment__ - Определяет что только фрагмент IP пакета будет заматчен. Когда пакеты больше чем максимальный размер с которым может справиться твоя система (это называется Maximum Transmission Unit или MTU), то они делятся на части и отправляются один за одним как одиночные пакеты. Смотри ifconfig(8) если хочешь узнать MTU своей системы (обычно по умолчанию это 1500 bytes).  
  Фрагменты часто используются в DOS аттаках потому что нет пути поиска происхождения фрмагмента пакета.  
- __sport|dport \[port-spec\]__ - Срабатывает на пакеты на указанный TCP или UDP порт. "sport" срабатывает на исходящий порт и "dport" срабатывает на конечный порт.  
  Эта проверка может быть использована только после указания "protocol tcp" или "protocol udp", потому что только в этих двух протоколах на данный момент есть порты.  
  И несколько примеров валидных портов/диапазонов:  
  ```
  dport 80 ACCEPT;
  dport http ACCEPT;
  dport ssh:http ACCEPT;
  dport 0:1023 ACCEPT; # equivalent to :1023
  dport 1023:65535 ACCEPT;
  ```
- __syn__ - Указывает что SYN флаг, который используется для создания новых TCP соединений, должен быть в TCP пакете. Так ты можешь идентифицировать входящие соединения и принимать решение разрешать их или нет. Пакеты не имеющие такого флага обычно из уже существующих соединений, поэтому считается достаточно безопасным пропускать их.  
- __module \[module-name\]__ - Загружает какой-либо iptables модуль. Множество модулей предоставляет много критериев. Мы вернемся к этому позже.  
  Вместо __module__ ты можешь также использовать сокращение __mod__  

#### Basic target keywords  
- __jump \[custom-chain-name\]__ - Перейти к кастомной цепочке. Если в кастомной цепочке нет походящего правила, netfilter вернется к следующему правилу в предыдущей цепочке.  
- __goto \[custom-chain-name\]__ - Перейти к кастомной цепочке. Не похоже на __jump__, __RETURN__ не будет продолжать обработку в этой цепочке, вместо этого продолжит в цепочке которая вызвала эту цепочку через __jump__.  
- __ACCEPT__ - Допустить заматченный пакет.  
- __DROP__ - Откинуть заматченный пакет без дальнейших уведомлений.  
- __REJECT__ - Реджектить заматченные пакеты, то есть отправлять ICMP пакеты отправителю, в которых по умолчанию port-unreachable. Ты можешь определить иной ICMP тип.  
  ```
  REJECT; # default to icmp-port-unreachable
  REJECT reject-with icmp-net-unreachable;
  ```
  Вводи "iptables -j REJECT -h"  для справки.  
- __RETURN__ - Закончить в текущей цепочке и вернуться к вызвавшей цепочке (если "jump \[custom-chain-name\]" использован)  
- __NOP__ - Нет действий для всех.  

### ADDITIONAL KEYWORDS  
Netfilter модульный. Модули погут предоставлять дополнительные критерии. Список модулей netfilter постоянно растет, а ferm пытается сохранить поддержку их всех.  

#### iptables match modules  
- __account__ -  Учитывать трафик для всех хостов в определенной сети. Это один из модулей поиска совпадений, который уже ведет себя как цель, поэтому в основном тебе придется использовать цель __NOP__.  
  ```
  mod account aname mynetwork aaddr 192.168.1.0/24 ashort NOP;
  ```
- __addrtype__ - Проверка типа адреса; любой: исходящй адрес или адрес назначения.  
  ```
  mod addrtype src-type BROADCAST;
  mod addrtype dst-type LOCAL;
  ```
  Набери "iptables -m addrtype -h" для справки.  
- __ah__ - Проверка SPI заголовка в AH пакете.  
  ```
  mod ah ahspi 0x101;
  mod ah ahspi ! 0x200:0x2ff;
  ```
  Дополнительные аргументы для IPv6:  
  ```
  mod ah ahlen 32 ACCEPT;
  mod ah ahlen !32 ACCEPT;
  mod ah ahres ACCEPT;
  ```
- __bpf__ - Матчинг с использованием Linux Socket Filter.  
  ```
  mod bpf bytecode "4,48 0 0 9,21 0 1 6,6 0 0 1,6 0 0 0";
  ```
- __cgroup__ - Матчинг  с использованием cgroupsv2 hierarchy или устаревшего net_cls cgroup.  
  ```
  mod cgroup path ! example/path ACCEPT;
  ```
  Путь относителен корня cgroupsv2 hierarchy и сравнивается с начальной частью пути процесса в иерархии.  
  ```
  mod cgroup cgroup 10:10 DROP;
  mod cgroup cgroup 1048592 DROP;
  ```
  Соответствует значению `net_cls.classid` установленному на процесс старым net_cls cgroup. Этот класс может быть определен как шестнадцатеричная major:minor пара (см. tc(8)), или как десятичное число, эти два правила эквивалентны.  
- __comment__ - Добавляет комментарий к правилу, не больше 256 символов, комментарий не имеет эффекта на пакет. Отметь что это не то же что и ferm комментарии ('#'), этот будет показан в "iptables -L".  
  ```
  mod comment comment "This is my comment." ACCEPT;
  ```
  "mod comment" можно не писать, ferm добавить автоматически.  
- __condition__ - Срабатывает если значение в /proc/net/ipt_condition/NAME равно 1 (путь для ip6 /proc/net/ip6t_condition/NAME).  
  ```
  mod condition condition (abc def) ACCEPT;
  mod condition condition !foo ACCEPT;
  ```
- __connbytes__ - Сопоставляет как много байт или пакетов было передано соединением до текущего момента, или среднее bytes per packet.  
  ```
  mod connbytes connbytes 65536: connbytes-dir both connbytes-mode bytes ACCEPT;
  mod connbytes connbytes !1024:2048 connbytes-dir reply connbytes-mode packets ACCEPT;
  ```
  Валидные значения для _connbytes-dir: original, reply, both_; для _connbytes-mode: packets, bytes, avgpkt_.  
- __connlabel__ - Модуль для проверки или добавления connlabel к соединению.  
  ```
  mod connlabel label "name";
  mod connlabel label "name" set;
  ```
- __connlimit__ - Позволяет тебе ограничивать количество параллельных TCP соединений к серверу по клиентскому IP адресу (или блоку адресов).  
  ```
  mod connlimit connlimit-above 4 REJECT;
  mod connlimit connlimit-above !4 ACCEPT;
  mod connlimit connlimit-above 4 connlimit-mask 24 REJECT;
  mod connlimit connlimit-upto 4 connlimit-saddr REJECT;
  mod connlimit connlimit-above 4 connlimit-daddr REJECT;
  ```
- __connmark__ - Проверка метки связанной с этим соединением, установленной CONNMARK'ом.  
  ```
  mod connmark mark 64;
  mod connmark mark 6/7;
  ```
- __conntrack__ - Проверка информации от механизма определения состояния соединения.  
  ```
  mod conntrack ctstate (ESTABLISHED RELATED);
  mod conntrack ctproto tcp;
  mod conntrack ctorigsrc 192.168.0.2;
  mod conntrack ctorigdst 1.2.3.0/24;
  mod conntrack ctorigsrcport 67;
  mod conntrack ctorigdstport 22;
  mod conntrack ctreplsrc 2.3.4.5;
  mod conntrack ctrepldst ! 3.4.5.6;
  mod conntrack ctstatus ASSURED;
  mod conntrack ctexpire 60;
  mod conntrack ctexpire 180:240;
  ```
  Введи "iptables -m conntrack -h" для справки.  
- __cpu__ - Сопоставление CPU обрабатывающего это соединение.  
  ```
  mod cpu cpu 0;
  ```
- __dccp__ - Проверка атрибутов специфичных для DCCP (Datagram Congestion Control Protocol). Этот модуль автоматически подгружается когда ты используешь "protocol dccp".  
  ```
  proto dccp sport 1234 dport 2345 ACCEPT;
  proto dccp dccp-types (SYNCACK ACK) ACCEPT;
  proto dccp dccp-types !REQUEST DROP;
  proto dccp dccp-option 2 ACCEPT;
  ```
- __dscp__ - Сопоставление 6-битного DSCP поля с TOS полем.  
  ```
  mod dscp dscp 11;
  mod dscp dscp-class AF41;
  ```
- __dst__ - Сравнение параметров в заголовке Destination Options (IPv6).  
  ```
  mod dst dst-len 10;
  mod dst dst-opts (type1 type2 ...);
  ```
- __ecn__ - Сравнение ECN битов IPv4 TCP заголовков.  
  ```
  mod ecn ecn-tcp-cwr;
  mod ecn ecn-tcp-ece;
  mod ecn ecn-ip-ect 2;
  ```
  Набери "iptables -m ecn -h" для подробной информации.  
- __esp__ - Сравнение SPI заголовка в ESP пакете.  
  ```
  mod esp espspi 0x101;
  mod esp espspi ! 0x200:0x2ff;
  ```
- __eui64__ - "Этот модуль матчит EUI-64 часть автонастраиваемого IPv6 адреса без сохранения состояния. Он сравнивает EUI-64 полученный из MAC адреса в Ethernet фрейме с наименьшими 64 битами исходящего IPv6 адреса. Но "Universal/Local" биты не сравниваются. Этот модуль не проверяет другие фреймы канального уровня, а только валидные в PREROUTING, INPUT и FORWARD цепочках."  
  ```
  mod eui64 ACCEPT;
  ```
- __fuzzy__ - "Этот модуль матчит пакеты которые входят в ограничение скорости основываясь на FLC."  
  ```
  mod fuzzy lower-limit 10 upper-limit 20 ACCEPT;
  ```
- __geoip__ - Матчит пакеты основываясь на их геопозиции. (Нужно установить GeoDB.)  
  ```
  mod geoip src-cc "CN,VN,KR,BH,BR,AR,TR,IN,HK" REJECT;
  mod geoip dst-cc "DE,FR,CH,AT" ACCEPT;
  ```
- __hbh__ - Матчит заголовок параметров Hop-by-Hop (ip6).  
  ```
  mod hbh hbh-len 8 ACCEPT;
  mod hbh hbh-len !8 ACCEPT;
  mod hbh hbh-opts (1:4 2:8) ACCEPT;
  ```
- __hl__ - Матчит поле Hop Limit (ip6).  
  ```
  mod hl hl-eq (8 10) ACCEPT;
  mod hl hl-eq !5 ACCEPT;
  mod hl hl-gt 15 ACCEPT;
  mod hl hl-lt 2 ACCEPT;
  ```
- __helper__ - Проверяет какой именно conntrack хелпер затрекал это соединение. Порт может быть указан через "-portnr".  
  ```
  mod helper helper irc ACCEPT;
  mod helper helper ftp-21 ACCEPT;
  ```
- __icmp__ - Проверяет специфичные для ICMP аттрибуты. Этот модуль автоматически подгружается когда ты используешь "protocol icmp".  
  ```
  proto icmp icmp-type echo-request ACCEPT;
  ```
  Эта опция может быть использована в _ip6_ домене, несмотря на то что в _ip6tables_ она называется __icmpv6__.  
  Используй "iptables -p icmp `-h`" чтобы получить список валидных ICMP типов.  
- __iprange__ - Матчит диапазон IPv4 адресов.  
  ```
  mod iprange src-range 192.168.2.0-192.168.3.255;
  mod iprange dst-range ! 192.168.6.0-192.168.6.255;
  ```
- __ipv4options__ - Проверяет в IPv4 заголовках такие опции как source routing, record route, timestamp and router-alert.  
  ```
  mod ipv4options ssrr ACCEPT;
  mod ipv4options lsrr ACCEPT;
  mod ipv4options no-srr ACCEPT;
  mod ipv4options !rr ACCEPT;
  mod ipv4options !ts ACCEPT;
  mod ipv4options !ra ACCEPT;
  mod ipv4options !any-opt ACCEPT;
  ```
- __ip6header__ - Матчит заголовки расширений IPv6 (ip6).  
  ```
  mod ipv6header header !(hop frag) ACCEPT;
  mod ipv6header header (auth dst) ACCEPT;
  ```
- __hashlimit__ - Похоже на 'mod limit', но добавляет способность добавления per-destination или per-port ограничений управляемых через hash таблицу.  
  ```
  mod hashlimit  hashlimit 10/minute  hashlimit-burst 30/minute
    hashlimit-mode dstip  hashlimit-name foobar  ACCEPT;
  ```
  Возможные значения для hashlimit-mode: dstip dstport srcip srcport (или список нескольких из этих значений).  
  Здесь больше возможных настроек, набери "iptables -m hashlimit -h" для просмотра документации.  
- __ipvs__ - Сравнивает свойства IPVS подключения.  
  ```
  mod ipvs ipvs ACCEPT; # packet belongs to an IPVS connection
  mod ipvs vproto tcp ACCEPT; # VIP protocol to match; by number or name, e.g. "tcp
  mod ipvs vaddr 1.2.3.4/24 ACCEPT; # VIP address to match
  mod ipvs vport http ACCEPT; # VIP port to match
  mod ipvs vdir ORIGINAL ACCEPT; # flow direction of packet
  mod ipvs vmethod GATE ACCEPT; # IPVS forwarding method used
  mod ipvs vportctl 80; # VIP port of the controlling connection to match
  ```
- __length__ - Проверяет длину пакета.  
  ```
  mod length length 128; # exactly 128 bytes
  mod length length 512:768; # range
  mod length length ! 256; # negated
  ```
- __limit__ - Ограничивает скорость пакетов.  
  ```
  mod limit limit 1/second;
  mod limit limit 15/minute limit-burst 10;
  ```
  Введи "iptables -m limit -h" для справки.  
- __mac__ - Сравнивает исходящий MAC адрес.  
  ```
  mod mac mac-source 01:23:45:67:89;
  ```
- __mark__ - Матчит пакеты основываясь на отметках netfilter mark. Здесь может быть 32 битное целое число между 0 и 4294967295.  
  ```
  mod mark mark 42;
  ```
- __mh__ - Матчит заголовки мобильности (ip6).  
  ```
  proto mh mh-type binding-update ACCEPT;
  ```
- __multiport__ - Матчит по набору исходящих или конечных портов (только UDP и TCP).  
  ```
  mod multiport source-ports (https ftp);
  mod multiport destination-ports (mysql domain);
  ```
  Это правило имеет большое преимущество над "dport" и "sport": оно генерирует только одно правило для вплоть до 15 портов вместо одного правила на каждый порт.  
  Как сокращение ты можешь использовать "sports" и "dports" (без "mod multiport"):  
  ```
  sports (https ftp);
  dports (mysql domain);
  ```
- __nth__ - Матчит каждый n-ный пакет.  
  ```
  mod nth every 3;
  mod nth counter 5 every 2;
  mod nth start 2 every 3;
  mod nth start 5 packet 2 every 6;
  ```
  Введи "iptables -m nth -h" для справки.  
- __osf__ - Матчит пакеты в зависимости от ОС отправителя.  
  ```
  mod osf genre Linux;
  mod osf ! genre FreeBSD ttl 1 log 1;
  ```
  Введи "iptables -m osf -h" для справки.  
- __owner__ - Проверяет информацию о создателе пакета, а именно user id, group id, process id, session id и command name.  
  ```
  mod owner uid-owner 0;
  mod owner gid-owner 1000;
  mod owner pid-owner 5432;
  mod owner sid-owner 6543;
  mod owner cmd-owner "sendmail";
  ```
  ("cmd-owner", "pid-owner" and "sid-owner" требуют специальных патчей для ядра, которые не включены в обычную поставку ядра Linux)  
- __physdev__ - Матчит пакеты по девайсу с/на которые пакет пришел/ушел. Это полезно для bridge-интерфейсов.  
  ```
  mod physdev physdev-in ppp1;
  mod physdev physdev-out eth2;
  mod physdev physdev-is-in;
  mod physdev physdev-is-out;
  mod physdev physdev-is-bridged;
  ```
- __pkttype__ - Проверяет тип пакета канального уровня.  
  ```
  mod pkttype pkt-type unicast;
  mod pkttype pkt-type broadcast;
  mod pkttype pkt-type multicast;
  ```
- __policy__ - Проверяет IPsec политику которая была применена к пакету.  
  ```
  mod policy dir out pol ipsec ACCEPT;
  mod policy strict reqid 23 spi 0x10 proto ah ACCEPT;
  mod policy mode tunnel tunnel-src 192.168.1.2 ACCEPT;
  mod policy mode tunnel tunnel-dst 192.168.2.1 ACCEPT;
  mod policy strict next reqid 24 spi 0x11 ACCEPT;
  ```
  Заметь что ключевое слово _proto_ также используется как сокращенная версия для _protocol_ (встроенные match module). Ты можешь исправить этот конфликт используя всегда длинное ключевое слово _protocol_.  
- __psd__ - Детектит TCP/UDP скан портов.  
  ```
  mod psd psd-weight-threshold 21 psd-delay-threshold 300
    psd-lo-ports-weight 3 psd-hi-ports-weight 1 DROP;
  ```
- __quota__ - Реализует сетевые квоты уменьшая счетчик байтов с каждым пакетом.  
  ```
  mod quota quota 65536 ACCEPT;
  ```
- __random__ - Матчит случайный процент всех пакетов.  
  ```
  mod random average 70;
  ```
- __realm__ - Матчит области маршрутизации. Полезно в окружениях использующих BGP.  
  ```
  mod realm realm 3;
  ```
- __recent__ - Временно отмечат исходящие IP адреса.  
  ```
  mod recent set;
  mod recent rcheck seconds 60;
  mod recent set rsource name "badguy";
  mod recent set rdest;
  mod recent rcheck rsource name "badguy" seconds 60;
  mod recent update seconds 120 hitcount 3 rttl;
  mod recent mask 255.255.255.0 reap;
  ```
  Этот модуль имеет конструкционный недостаток: несмотря на то что он реализован как match module, он имеет target-like поведение когда используется ключевое слово "set".  
  <http://snowman.net/projects/ipt_recent/>  
- __rpfilter__ - Проверяет, что ответ на пакет будет отправлен через тот же интерфейс, с которого он прибыл. Пакеты из loopback интерфейса всегда ограничиваются.  
  ```
  mod rpfilter proto tcp loose RETURN;
  mod rpfilter validmark accept-local RETURN;
  mod rpfilter invert DROP;
  ```
  Этот модуль является предпочтительным способом выполнения фильтрации обратного пути для IPv6 и мощной альтернативой проверкам, контролируемым через sysctl _net.ipv4.conf.*.rp\_filter_.  
- __rt__ - Матчит IPv6 роутинг заголовки (ip6).  
  ```
  mod rt rt-type 2 rt-len 20 ACCEPT;
  mod rt rt-type !2 rt-len !20 ACCEPT;
  mod rt rt-segsleft 2:3 ACCEPT;
  mod rt rt-segsleft !4:5 ACCEPT;
  mod rt rt-0-res rt-0-addrs (::1 ::2) rt-0-not-strict ACCEPT;
  ```
- __sctp__ - Проверят специфичные для SCTP (Stream Control Transmission Protocol) аттрибуты. Этот модуль автоматически подгружается когда ты используешь "protocol sctp".  
  ```
  proto sctp sport 1234 dport 2345 ACCEPT;
  proto sctp chunk-types only DATA:Be ACCEPT;
  proto sctp chunk-types any (INIT INIT_ACK) ACCEPT;
  proto sctp chunk-types !all (HEARTBEAT) ACCEPT;
  ```
  Используй "iptables -p sctp `-h`" чтобы показать список валидных типов.  
- __set__ - Сравнивает исходящий или конечный IP/Port/MAC с указанным набором.  
  ```
  mod set set badguys src DROP;
  ```
  Смотри <http://ipset.netfilter.org/> для большей информации.  
- __state__ - Проверяет состояние соединения.  
  ```
  mod state state INVALID DROP;
  mod state state (ESTABLISHED RELATED) ACCEPT;
  ```
  Введи "iptables -m state -h" для справки.  
- __statistic__ - Преемник __nth__ и __random__, на данный момент недокументирован в iptables(8) man page.  
  ```
  mod statistic mode random probability 0.8 ACCEPT;
  mod statistic mode nth every 5 packet 0 DROP;
  ```
- __string__ - Сравнивает строку.  
  ```
  mod string string "foo bar" ACCEPT;
  mod string algo kmp from 64 to 128 hex-string "deadbeef" ACCEPT;
  ```
- __tcp__ - Проверяет специфичные для TCP аттрибуты. Этот модуль автоматически подгружается когда ты используешь "protocol tcp".  
  ```
  proto tcp sport 1234;
  proto tcp dport 2345;
  proto tcp tcp-flags (SYN ACK) SYN;
  proto tcp tcp-flags ! (SYN ACK) SYN;
  proto tcp tcp-flags ALL (RST ACK);
  proto tcp syn;
  proto tcp tcp-option 2;
  proto tcp mss 512;
  ```
  Набери "iptables -p tcp -h" для справки.  
- __tcpmss__ - Проверяет поле TCP MSS в SYN или SYN/ACK пакетах.  
  ```
  mod tcpmss mss 123 ACCEPT;
  mod tcpmss mss 234:567 ACCEPT;
  ```
- __time__ - Сравнивает время поступления пакета с указанными диапазонами.  
  ```
  mod time timestart 12:00;
  mod time timestop 13:30;
  mod time timestart 22:00 timestop 07:00 contiguous;
  mod time days (Mon Wed Fri);
  mod time datestart 2005:01:01;
  mod time datestart 2005:01:01:23:59:59;
  mod time datestop 2005:04:01;
  mod time monthday (30 31);
  mod time weekdays (Wed Thu);
  mod time timestart 12:00;
  mod time timestart 12:00 kerneltz;
  ```
  Набери "iptables -m time -h" для справки.  
- __tos__ - Матчит пакеты по указанному TOS значению.  
  ```
  mod tos tos Minimize-Cost ACCEPT;
  mod tos tos !Normal-Service ACCEPT;
  ```
  Набери "iptables -m tos -h" для справки.  
- __ttl__ - Матчит по TTL (time to live) полю в IP заголовке.  
  ```
  mod ttl ttl-eq 12; # ttl equals
  mod ttl ttl-gt 10; # ttl greater than
  mod ttl ttl-lt 16; # ttl less than
  ```
- __u32__ - Сравнивает сырые данные из пакета. Ты можешь указать больше чем один фильтр через список; они не развернутся в множество правил.  
  ```
  mod u32 u32 '6&0xFF=1' ACCEPT;
  mod u32 u32 ('27&0x8f=7' '31=0x527c4833') DROP;
  ```
- __unclean__ - Матчит пакеты которые выглядят как бесформенные или необычные. Эта проверка не имеет дополнительных параметров.  

#### iptables target modules  
Следующие дополнительные цели доступны в ferm, при условии что ты включил их в своем ядре:  
- __CHECKSUM__ - Считает чек-сумму пакета.  
  ```
  CHECKSUM checksum-fill;
  ```
- __CLASSIFY__ - Устанавливат CBQ класс.  
  ```
  CLASSIFY set-class 3:50;
  ```
- __CLUSTERIP__ - Конфигурирует простой кластер нод которые делят определенный IP и MAC адрес. Соединение статически доставляется между нодами.  
  ```
  CLUSTERIP new hashmode sourceip clustermac 00:12:34:45:67:89
    total-nodes 4 local-node 2 hash-init 12345;
  ```
- __CONNMARK__ - Устанавливает netfilter отметки связанные с этим соединением.  
  ```
  CONNMARK set-xmark 42/0xff;
  CONNMARK set-mark 42;
  CONNMARK save-mark;
  CONNMARK restore-mark;
  CONNMARK save-mark nfmask 0xff ctmask 0xff;
  CONNMARK save-mark mask 0x7fff;
  CONNMARK restore-mark mask 0x8000;
  CONNMARK and-mark 0x7;
  CONNMARK or-mark 0x4;
  CONNMARK xor-mark 0x7;
  CONNMARK and-mark 0x7;
  ```
- __CONNSECMARK__ - Этот модуль копирует маркировки безопасности из пакета на соединение (если не помечено), и из соединения возвращает пакеты (также, только если не помечены). Обычно используется в связи с SECMARK, он доступен только для таблицы mangle.  
  ```
  CONNSECMARK save;
  CONNSECMARK restore;
  ```
- __DNAT to \[ip-address|ip-range|ip-port-range\]__ - Изменяет адрес назначения пакета.  
  ```
  DNAT to 10.0.0.4;
  DNAT to 10.0.0.4:80;
  DNAT to 10.0.0.4:1024-2048;
  DNAT to 10.0.1.1-10.0.1.20;
  ```
- __DNPT__ - Предоставляет IPv6-to-IPv6 Трансляцию Сетевых Префиксов без сохранения состояния назначения.  
  ```
  DNPT src-pfx 2001:42::/16 dst-pfx 2002:42::/16;
  ```
- __ECN__ - Позволяет выборочно работать вокруг известных черных дыр ECN. Доступно только в таблице mangle.  
  ```
  ECN ecn-tcp-remove;
  ```
- __HL__ - Изменяет поле IPv6 Hop Limit (ip6/mangle).  
  ```
  HL hl-set 5;
  HL hl-dec 2;
  HL hl-inc 1;
  ```
- __HMARK__ - Как MARK, т.е. устанавливает fwmark, но метка вычисляется из селектора пакетов хэширования по выбору.  
  ```
  HMARK hmark-tuple "src" hmark-mod "1" hmark-offset "1"
    hmark-src-prefix 192.168.1.0/24 hmark-dst-prefix 192.168.2.0/24
    hmark-sport-mask 0x1234 hmark-dport-mask 0x2345
    hmark-spi-mask 0xdeadbeef hmark-proto-mask 0x42 hmark-rnd 0xcoffee;
  ```
- __IDLETIMER__ - Может быть использовано для идентификации когда интерфейс проставивает в определенный период времени.  
  ```
  IDLETIMER timeout 60 label "foo";
  ```
- __IPV4OPTSSTRIP__ - Очищает все IP опции с пакета. Модуль не имеет опций.  
  ```
  IPV4OPTSSTRIP;
  ```
- __JOOL__ - Передает пакеты для стэйтфул NAT64 трансляции через JOOL. JOOL должен быть установлен в системе и модуль ядра jool дожен быть загружен.  
  ```
  JOOL instance "foo";
  ```
- __JOOL_SIIT__ - Передает пакеты для стэйтлес IP/ICMP трансляции (SIIT) через JOOL. JOOL должен быть установлен в системе и модуль ядра jool_siit дожен быть загружен.  
  ```
  JOOL_SIIT instance "foo";
  ```
- __LED__ - Создает LED-триггер который может быть приаттачен к панели индикаторов, чтобы мигать или светить ими когда определенные пакеты проходят через систему.  
  ```
  LED led-trigger-id "foo" led-delay 100 led-always-blink;
  ```
- __LOG__ - Логирует пакеты которые матчатся этим правилом в kernel log. Будь осторожен с лог флудингом. Отметь что это "non-terminating target", т.е. обход по правилам будет продолжен.  
  ```
  LOG log-level warning log-prefix "Look at this: ";
  LOG log-tcp-sequence log-tcp-options;
  LOG log-ip-options;
  ```
- __MARK__ - Устанавливает netfilter маркировку для пакета (32 битное целое число между 0 и 4294967295):  
  ```
  MARK set-mark 42;
  MARK set-xmark 7/3;
  MARK and-mark 31;
  MARK or-mark 1;
  MARK xor-mark 12;
  ```
- __MASQUERADE__ - Маскарадит заматченные пакеты. Опционально за ним следует порт или диапазон портов для iptables. Указывается как "123", "123-456" или "123:456". Параметр с диапазоном портов определяет через какие локальные порты должно проходить замаскараженое соединение.  
  ```
  MASQUERADE;
  MASQUERADE to-ports 1234:2345;
  MASQUERADE to-ports 1234:2345 random;
  ```
- __MIRROR__ - Экспериментально/демонстрационная цель, которая меняет местами получателя и отправителя в IP заголовках.  
  ```
  MIRROR;
  ```
- __NETMAP__ - Мапит целую сеть на другую сеть в __nat__ таблице.  
  ```
  NETMAP to 192.168.2.0/24;
  ```
- __NOTRACK__ - Отключает conntrack для всех пакетов которые матчатся этим правилом.  
  ```
  proto tcp dport (135:139 445) NOTRACK;
  ```
- __RATEEST__  
  ```
  RATEEST rateest-name "foo" rateest-interval 60s rateest-ewmalog 100;

  proto tcp dport (135:139 445) NOTRACK;
  ```
- __NFLOG__ - Логирует пакеты по netlink; это наследник ULOG'a.  
  ```
  NFLOG nflog-group 5 nflog-prefix "Look at this: ";
  NFLOG nflog-range 256;
  NFLOG nflog-threshold 10;
  ```
- __NFQUEUE__ - Очерединг, требует поддержку ядром nfnetlink_queue.  
  ```
  proto tcp dport ftp NFQUEUE queue-num 20;
  ```
- __QUEUE__ - Очерединг, предок __NFQUEUE__. Все пакеты уходят в очередь 0.  
  ```
  proto tcp dport ftp QUEUE;
  ```
- __REDIRECT to-ports \[ports\]__ - Прозрачное проксирование: изменяет IP назначения пакета на свой.  
  ```
  proto tcp dport http REDIRECT to-ports 3128;
  proto tcp dport http REDIRECT to-ports 3128 random;
  ```
- __SAME__ - Похоже на SNAT, но клиент мапится на один и тот же исходящий IP для всех его подключений.  
  ```
  SAME to 1.2.3.4-1.2.3.7;
  SAME to 1.2.3.8-1.2.3.15 nodst;
  SAME to 1.2.3.16-1.2.3.31 random;
  ```
- __SECMARK__ - Используется для маркирования пакета для подсистемы безопасности такой как SELinux. Это работает только в таблице mangle.  
  ```
  SECMARK selctx "system_u:object_r:httpd_packet_t:s0";
  ```
- __SET \[add-set|del-set\] \[setname\] \[flag(s)\]__ - Добавляет IP в специальный список. См. <http://ipset.netfilter.org/>  
  ```
  proto icmp icmp-type echo-request SET add-set badguys src;
  SET add-set "foo" timeout 60 exist;
  ```
- __SNAT to \[ip-address|ip-range|ip-port-range\]__ - Изменяет исходящий адрес пакета.  
  ```
  SNAT to 1.2.3.4;
  SNAT to 1.2.3.4:20000-30000;
  SNAT to 1.2.3.4 random;
  ```
- __SNPT__ - Предоставляет IPv6-to-IPv6 трансляцию исходящих сетевых префиксов без сохранения состояния.  
  ```
  SNPT src-pfx 2001:42::/16 dst-pfx 2002:42::/16;
  ```
- __SYNPROXY__ - Проксирование тройного рукопожатия в TCP: пусть файерволл обрабатывает тройное рукопожатие TCP и устанавливает соединение с сокетом сервера единожды для тех клиентов которые прошли стадию рукопожатия.  
  ```
  SYNPROXY wscale 7 mss 1460 timestamp sack-perm
  ```
- __TCPMSS__ - Подменяет MSS значение в TCP SYN пакетах.  
  ```
  TCPMSS set-mss 1400;
  TCPMSS clamp-mss-to-pmtu;
  ```
- __TCPOPTSTRIP__ - Лишает TCP пакет его TCP опций.  
  ```
  TCPOPTSTRIP strip-options (option1 option2 ...);
  ```
- __TOS set-tos \[value\]__ - Устанавлиет TCP Type Of Service в указанное значение. Это будет использоваться любым планировщиком трафика, который захочет, в основном вашей собственной Linux-машиной, но, возможно, и больше. Оригинальные TOS-биты очищаются и перезаписываются.  
  ```
  TOS set-tos Maximize-Throughput;
  TOS and-tos 7;
  TOS or-tos 1;
  TOS xor-tos 4;
  ```
  Введи "iptables -j TOS -h" для справки.  
- __TTL__ - Изменяет заголовок TTL.  
  ```
  TTL ttl-set 16;
  TTL ttl-dec 1; # decrease by 1
  TTL ttl-inc 4; # increase by 4
  ```
- __ULOG__ - Логирует пакеты пользовательской программой.  
  ```
  ULOG ulog-nlgroup 5 ulog-prefix "Look at this: ";
  ULOG ulog-cprange 256;
  ULOG ulog-qthreshold 10;
  ```

### ДРУГИЕ ДОМЕНЫ  
Начиная с версии 2.0, __ferm__ поддерживает не только _ip_ и _ip6_, но также _arp_ (ARP таблицы) и _eb_ (ethernet bridging таблицы). Эти концепции аналогичны _iptables_.  

#### Ключевые слова arptables  
- __source-ip, destination-ip__ - Матчат исходящие или конечные IPv4 адреса. Похоже на __saddr__ и __daddr__ в _ip_ домене.  
- __source-mac, destination-mac__ - Матчат исходящие или конечные MAC адреса.  
- __interface, outerface__ - Входной или выходной интерфейс.  
- __h-length__ - Аппаратная длина пакета.  
  ```
  chain INPUT h-length 64 ACCEPT;
  ```
- __opcode__ - Код операции, для справки смотри iptables(8).  
  ```
  opcode 9 ACCEPT;
  ```
- __h-type__ - Аппаратный тип.  
  ```
  h-type 1 ACCEPT;
  ```
- __proto-type__ - Тип протокола.  
  ```
  proto-type 0x800 ACCEPT;
  ```
- __Mangling__ - Ключевые слова __mangle-ip-s, mangle-ip-d, mangle-mac-s, mangle-mac-d, mangle-target__ могут быть использованы для изменения ARP. См. iptables(8) для справки.  

#### Ключевые слова ebtables  
- __proto__ - Матчит протокол которым создан этот фрейм, например IPv4 или __PPP__. Для просмотра списка, см. _/etc/ethertypes_.  
- __interface, outerface__ - Физический входной или выходной интерфейс.  
- __logical-in, logical-out__ - Логический bridge интерфейс.  
- __saddr, daddr__ - Матчит исходный или конечный MAC адрес.  
- __Match modules__ - Поддерживаются следующие модули: 802.3, arp, ip, mark_m, pkttype, stp, vlan, log.  
- __Target extensions__ - Поддерживаются следующие расширения: arpreply, dnat, mark, redirect, snat.  
  Пожалуйста отметь что здесь происходит конфликт между _--mark_ из _mark\_m_ модуля и _-j mark_. Поскольку они оба сработают по ключевому слову __mark__, то мы решили эту проблему написанием имени цели в вернем регистре. Как и в других доменах. Следующий пример перезаписывает маркировку 1 на 2:  
  ```
  mark 1 MARK 2;
  ```

### ДОПОЛНИТЕЛЬНЫЕ ВОЗМОЖНОСТИ  
#### Переменные  
В сложных файервольных файлах полезно использовать переменные, например чтобы дать сетевому интерфейсу осмысленное имя.  

Чтобы указать переменную, напиши:  
```
@def $DEV_INTERNET = eth0;
@def $PORTS = (http ftp);
@def $MORE_PORTS = ($PORTS 8080);
```
В настоящем ferm коде, переменные используются как параметры:  
```
chain INPUT interface $DEV_INTERNET proto tcp dport $MORE_PORTS ACCEPT;
```
Заметь, что переменные могуть использоваться только в параметрах ("192.168.1.1", "http"); они не могут содержать ключевых слов как "proto" или "interface".  

Переменные действительны только в текущем блоке:  
```
@def $DEV_INTERNET = eth1;
chain INPUT {
    proto tcp {
        @def $DEV_INTERNET = ppp0;
        interface $DEV_INTERNET dport http ACCEPT;
    }
    interface $DEV_INTERNET DROP;
}
```
будет перобразован в:  
```
chain INPUT {
    proto tcp {
        interface ppp0 dport http ACCEPT;
    }
    interface eth1 DROP;
}
```
"def $DEV_INTERNET = ppp0" валидно только в блоке "proto tcp"; родительский блок остается с "set $DEV_INTERNET = eth1".  

Подключаемые файлы с описанными переменными остаются доступными в вызвавшем блоке. Это полезно когда ты инклудишь файлы в которых описаны только переменные.  

#### Автоматические переменные  
Некоторые переменные устанавливаются ferm'ом изнутри. Ferm скрипты могут использовать их просто как любые другие переменные.  

- __$FILENAME__ - Имя конфигурационного файла относительно директории откуда был запущен ferm.  
- __$FILEBNAME__ - Базовое имя конфигурационного файла.  
- __$DIRNAME__ - Директория конфигурационного файла.  
- __$DOMAIN__ - Текущий домен. Один из _ip_, _ip6_, _arp_, _eb_.  
- __$TABLE__ - Текущая netfilter таблица.  
- __$CHAIN__ - Текущая netfilter цепочка.  
- __$LINE__ - Строка текущего скрипта. Это может быть использовано например так:  
  ```
  @def &log($msg) = {
               LOG log-prefix "rule=$msg:$LINE ";
  }
  .
  .
  .
  &log("log message");
  ```

#### Функции  
Функции похожи на переменные, за исключением того что они могут принимать параметры и они предоставляют ferm'у команды а не значения.  
```
@def &FOO() = proto (tcp udp) dport domain;
&FOO() ACCEPT;

@def &TCP_TUNNEL($port, $dest) = {
    table filter chain FORWARD interface ppp0 proto tcp dport $port daddr $dest outerface eth0 ACCEPT;
    table nat chain PREROUTING interface ppp0 proto tcp dport $port daddr 1.2.3.4 DNAT to $dest;
}

&TCP_TUNNEL(http, 192.168.1.33);
&TCP_TUNNEL(ftp, 192.168.1.30);
&TCP_TUNNEL((ssh smtp), 192.168.1.2);
```
Вызов функции, который содержит блок (как '{...}') должен быть последней командой в ferm правиле, например за ним должна следовать ';'. '$FOO()' не содержит блок и поэтому ты можешь написать 'ACCEPT' после ее вызова. Для обхода этого ты можешь реорганизовать ключевые слова:  
```
@def &IPSEC() = { proto (esp ah); proto udp dport 500; }
chain INPUT ACCEPT &IPSEC();
```

#### Апострофы  
С апострофами ты можешь использовать вывод внешних команд:  
```
@def $DNSSERVERS = `grep nameserver /etc/resolv.conf | awk '{print $2}'`;
chain INPUT proto tcp saddr $DNSSERVERS ACCEPT;
```
Команды выполняются в shell (_/bin/sh_), просто как апострофы в Perl. Ferm не разворачивает никакие переменные в них.  

Затем вывод помечается и сохраняется как ferm список (массив). Строки начинающиеся с '#' игнорируются; другие строки могут содержать любое количество значений разделенных пробелом.  

#### Включения  
Ключевое слово __@include__ позволяет тебе включать внешние файлы:  
```
@include 'vars.ferm';
```
Имя файла относительно к вызывающему файлу, например выражение выше включает _/etc/ferm/vars.ferm_ когда включение происходит из _/etc/ferm/ferm.conf_.  
Переменные и функции описанные во включенном файле остаются доступными в вызвавшем файле.  

__include__ работает в блоках:  
```
chain INPUT {
    @include 'input.ferm';
}
```
Если ты укажешь директорию (с '/' на конце), все файлы из этой директории будут включены в алфавитном порядке:  
```
@include 'ferm.d/';
```
Функция @glob может быть использована для разворачивания вайлдкардов:  
```
@include @glob('*.include');
```
С пайп-символом на конце, __ferm__ выполняет shell команду и парсит вывод:  
```
@include "/root/generate_ferm_rules.sh $HOSTNAME|"
```
__ferm__ прервется если код возврата не 0.  

#### Условия  
Ключевое слово __@if__ предоставляет условное выражение:  
```
@if $condition DROP;
```
Значение расценивается как true также как в Perl: ноль, пустой список, пустая строка это ложь, все остальное это истина. Примеры для истинных значений:  
```
(a b); 1; 'foo'; (0 0)
```
Примеры для ложных значений:  
```
(); 0; '0'; ''
```
Также здесь есть __@else__:  
```
@if $condition DROP; @else REJECT;
```
Учитывай точку с запятой перед __@else__.  

Можно использовать фигурные скобки перед __@if__ и __@else__:  
```
@if $condition {
    MARK set-mark 2;
    RETURN;
} @else {
    MARK set-mark 3;
}
```
С закрывающейся фигурной скобкой также завершается команда, поэтому точка с запятой здесь не нужна.  

Здесь нет __@elsif__, используй вместо этого __@else @if__.  

Пример:  
```
@def $have_ipv6 = `test -f /proc/net/ip6_tables_names && echo 1 || echo`;
@if $have_ipv6 {
    domain ip6 {
        # ....
    }
}
```

#### Хуки  
Для запуска кастомных команд ты можешь устанавливать хуки:  
```
@hook pre "echo 0 >/proc/sys/net/ipv4/conf/eth0/forwarding";
@hook post "echo 1 >/proc/sys/net/ipv4/conf/eth0/forwarding";
@hook flush "echo 0 >/proc/sys/net/ipv4/conf/eth0/forwarding";
```
Описанные команды выполняются используя shell. "pre" значит запустить команду до применения файерволл правил, а "post" значит запустить команду после. "flush" хук запускается после того как ferm зафлашит файерволл правила (опция --flush). Ты можешь установить любое количество хуков.  

### Встроенные функции  
Здесь некоторые встроенные функции, которые ты можешь найти полезными.  

#### @defined($name), @defined(&name)  
Тестирует определена ли функция или переменная.  
```
@def $a = 'foo';
@if @defined($a) good;
@if @not(@defined($a)) bad;
@if @defined(&funcname) good;
```

#### @eq(a,b)  
Тестирует два значения на равенство. Например:  
```
@if @eq($DOMAIN, ip6) DROP;
```

#### @ne(a,b)  
Похоже на @eq, оно тестирует на неравенство.  

#### @not(x)  
Инвертирует логическое значение.  

#### @resolve((hostname1 hostname2 ...), \[type\])  
Обычно, хостнэймы резолвятся iptables'ом. Чтобы позволить ferm'у резолвить хостнэймы, используй функцию @resolve:  
```
saddr @resolve(my.host.foo) proto tcp dport ssh ACCEPT;
saddr @resolve((another.host.foo third.host.foo)) proto tcp dport openvpn ACCEPT;
daddr @resolve(ipv6.google.com, AAAA) proto tcp dport http ACCEPT;
```
Отметь двойные скобки на второй строке: внутренняя пара создает список, в внешняя как разделитель параметров функции.  

Второй параметр опционален и определяет тип DNS записи. По умолчанию это "A" для домена ip и "AAAA" для домена ip6.  

Будь аккуратен с зарезолвленными хостнэймами в конфигурации файерволла. DNS запросы могут блокировать конфигурирование файерволла на долгое время, оставляя машину уязвимой, или они могут фэйлиться.  

#### @cat(a, b, ...)  
Конкатенирует все параметры в одну строку.  

#### @join(separator, a, b, ...)  
Объединяет все параметры в одну строку разделяя указанным разделителем.  

#### @substr(expression, offset, length)  
Извлекает подстроку из выражения и возвращает ее. Первый символ имеет отступ 0. Если OFFSET отрицательный, то отсчитывается настолько же далеко от конца строки.  

#### @length(expression)  
Возвращает длину выражения в символах.  

#### @basename(path)  
Возвращает базовое имя файла для указанного пути (File::Spec::splitpath).  

#### @dirname(path)  
Возвращает имя последней директории для указанного пути, предполагая что последний компонент это имя файла (File::Spec::splitpath).  

#### @glob(path)  
Разворачивает shell wildcard'ы в указанных путях (предполагается что они относительны к текущему скрипту). Возвращает список заматченных файлов. Эта функция полезна как параметр для @include.  

#### @ipfilter(list)  
Отфильтровывает IP адреса которые, очевидно, не подходят под текущий домен. Это полезно для создания общих переменных и правил для IPv4 и IPv6:  
```
@def $TRUSTED_HOSTS = (192.168.0.40 2001:abcd:ef::40);

domain (ip ip6) chain INPUT {
    saddr @ipfilter($TRUSTED_HOSTS) proto tcp dport ssh ACCEPT;
}
```

### РЕЦЕПТЫ  
Директория _./examples/_ содержит многочисленные ferm конфигурации, которые могут быть использованы для начала в новом файерволле. Эта секция содержит много примеров, рецептов и трюков.  

#### Легкий проброс портов  
Ferm функции делают рутинные задачи быстрыми и легкими:  
```
@def &FORWARD_TCP($proto, $port, $dest) = {
    table filter chain FORWARD interface $DEV_WORLD outerface $DEV_DMZ daddr $dest proto $proto dport $port ACCEPT;
    table nat chain PREROUTING interface $DEV_WORLD daddr $HOST_STATIC proto $proto dport $port DNAT to $dest;
}

&FORWARD_TCP(tcp, http, 192.168.1.2);
&FORWARD_TCP(tcp, smtp, 192.168.1.3);
&FORWARD_TCP((tcp udp), domain, 192.168.1.4);
```

#### Удаленный ferm  
Если удаленная машина не способна запустить __ferm__ по каким-то причинам  (может быть не имеется Perl), то ты можешь редактировать конфигурационный файл __ferm__ на другом компьютере и дать __ferm__'у сгенерировать shell скрипт.  

Пример для OpenWRT:  
```
ferm --remote --shell mywrt/ferm.conf >mywrt/firewall.user
chmod +x mywrt/firewall.user
scp mywrt/firewall.user mywrt.local.net:/etc/
ssh mywrt.local.net /etc/firewall.user
```

### ОПЦИИ  
- __--noexec__ - Не выполнят iptables(8) команды, а пропускать их. С помощью этого ты можешь парсить свои данные, используй __--lines__ для отображения вывода.  
- __--flush__ - Очистить файерволл правила и установить политики всех цепочек в ACCEPT. __ferm__'у потребуется конфигурационный файл чтобы понять какие домены и таблицы пострадают.  
- __--lines__ - Отображать строки которые были сгенерированы из правил. Они будут показаны перед тем как будут исполнены, поэтому если ты получишь сообщение об ошибке из iptables(8) итд, то ты сможешь увидеть какое правило вызвало ошибку.  
- __--interactive__ - Применять правила и спрашивать подтверждение у пользователя. Отменяет предыдущий набор правил если не было пользовательского ответа в течение 30 секунд (см. __---timeout__). Это полезно для удаленного администрирования файерволла: ты можешь тестировать правила без страха заблокировать себе доступ.  
- __--timeout S__ - Если используется __--interactive__, то делает откат если в течение указанного количества секунд не было валидного пользовательского ответа. По умолчанию 30.  
- __--help__ - Показывает краткий список доступных опций.  
- __--version__ - Показывает номер версии программы.  
- __--fast__ - Включает быстрый режим: ferm генерирует iptables-save(8) файл и устанавливает его через iptables-restore(8). Это намного быстрее, потому что по умолчанию ferm вызывает iptables(8) по одному разу на каждое правило.  
  Быстрый режим включен по умолчанию начиная с __ferm__ 2.0, и эта опция задепрекейчена.  
- __--slow__ - Отключает быстрый режим, т.е. запускает iptables(8) для каждого правила и не использует iptables-restore(8).  
- __--shell__ - Генерирует shell скрипт, который вызывает iptables-restore(8) и печатает. Заключает в себе --fast и --lines.  
- __--remote__ - Генерирует правила для удаленной машины. Подразумевает __--noexec__ и __--lines__. Может быть скомбинирован с __--shell__.  
- __--domain {ip|ip6}__ - Обрабатывает только указанный домен и выбирает его как домен по умолчанию для правил где он не определен. Вывод __ferm__ может быть пустым если домен не сконфигурирован во входном файле.  
- __--def '$name=value'__ - Переопределяет переменные определенные в конфигрурационном файле.  

### СМОТРИ ТАКЖЕ  
iptables(8)  

### ТРЕБОВАНИЯ  
#### Операционная система  
Linux 2.4 или новее, с поддержкой netfilter и всеми модулями которые используются твоими скриптами  

#### Программное обеспечение  
iptables и perl 5.6  

### БАГИ  
Баги? Какие еще баги?  
Если ты нашел баг, пожалуйста уведоми об этом на GitHub: <https://github.com/MaxKellermann/ferm/issues>  

### КОПИРАЙТ  
Copyright 2001-2017 Max Kellermann <max.kellermann@gmail.com>, Auke Kok <sofar@foo-projects.org> and various other contributors.  

This program is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 2 of the License, or (at your option) any later version.  

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.  

You should have received a copy of the GNU General Public License along with this program; if not, write to the Free Software Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.  

### АВТОР  
Max Kellermann <max.kellermann@gmail.com>, Auke Kok <sofar@foo-projects.org>

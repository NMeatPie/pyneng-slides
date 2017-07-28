# Python для сетевых инженеров 

#HSLIDE

# Шаблоны конфигураций с Jinja

#VSLIDE
### Шаблоны конфигураций с Jinja

[Jinja2](http://xgu.ru/wiki/Jinja2) это язык шаблонов, который используется в Python.

Jinja2 используется для генерации документов на основе одного или нескольких шаблонов.

Примеры использования:
* шаблоны для генерации HTML-страниц
* шаблоны для генерации конфигурационных файлов в Unix/Linux
* шаблоны для генерации конфигурационных файлов сетевых устройств


Установить Jinja2 можно с помощью pip:
```python
pip install jinja2
```

#VSLIDE
### Шаблоны конфигураций с Jinja

Идея Jinja очень проста: разделение данных и шаблона.
Это позволяет использовать один и тот же шаблон, но подставлять в него разные данные.

В самом простом случае, шаблон это просто текстовый файл, в котором указаны места подстановки значений, с помощью переменных Jinja.

#VSLIDE
### Шаблоны конфигураций с Jinja

Пример шаблона Jinja:
```jinja
hostname {{name}}
!
interface Loopback255
 description Management loopback
 ip address 10.255.{{id}}.1 255.255.255.255
!
interface GigabitEthernet0/0
 description LAN to {{name}} sw1 {{int}}
 ip address {{ip}} 255.255.255.0
!
router ospf 10
 router-id 10.255.{{id}}.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
```

#VSLIDE
### Шаблоны конфигураций с Jinja

Пример скрипта с генерацией файла на основе шаблона Jinja (файл basic_generator.py):
```python
from jinja2 import Template

template = Template("""
hostname {{name}}
!
interface Loopback255
 description Management loopback
 ip address 10.255.{{id}}.1 255.255.255.255
!
interface GigabitEthernet0/0
 description LAN to {{name}} sw1 {{int}}
 ip address {{ip}} 255.255.255.0
!
router ospf 10
 router-id 10.255.{{id}}.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
""")

liverpool = {'id':'11', 'name':'Liverpool', 'int':'Gi1/0/17', 'ip':'10.1.1.10'}

print(template.render(liverpool))
```

#VSLIDE
### Шаблоны конфигураций с Jinja

Если запустить скрипт basic_generator.py, то вывод будет таким:
```
$ python basic_generator.py

hostname Liverpool
!
interface Loopback255
 description Management loopback
 ip address 10.255.11.1 255.255.255.255
!
interface GigabitEthernet0/0
 description LAN to Liverpool sw1 Gi1/0/17
 ip address 10.1.1.10 255.255.255.0
!
router ospf 10
 router-id 10.255.11.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
```

 
#HSLIDE

## Пример использования Jinja2

#VSLIDE
### Пример использования Jinja2

В этом примере логика разнесена в 3 разных файла (все файлы находятся в каталоге 1_example):
* router_template.py - шаблон
* routers_info.yml - в этом файле, в виде списка словарей (в формате YAML), находится информация о маршрутизаторах, для которых нужно сгенерировать конфигурационный файл
* router_config_generator.py - в этом скрипте импортируется файл с шаблоном и считывается информация из файла в формате YAML, а затем генерируются конфигурационные файлы маршрутизаторов

#VSLIDE
### Пример использования Jinja2

Файл router_template.py
```python
# -*- coding: utf-8 -*-
from jinja2 import Template

template_r1 = Template("""
hostname {{name}}
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.{{id}}.1 255.255.255.255
 !
interface GigabitEthernet0/0
 description WAN to {{name}} sw1 G0/1
!
interface GigabitEthernet0/0.1{{id}}1
 description MPLS to {{to_name}}
 encapsulation dot1Q 1{{id}}1
 ip address 10.{{id}}.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN {{name}} to sw1 G0/2 !
interface GigabitEthernet0/1.{{IT}}
 description PW IT {{name}} - {{to_name}}
 encapsulation dot1Q {{IT}}
 xconnect 10.10.{{to_id}}.1 {{id}}11 encapsulation mpls
 backup peer 10.10.{{to_id}}.2 {{id}}21
  backup delay 1 1
!
interface GigabitEthernet0/1.{{BS}}
 description PW BS {{name}} - {{to_name}}
 encapsulation dot1Q {{BS}}
 xconnect 10.10.{{to_id}}.1 {{to_id}}{{id}}11 encapsulation mpls
  backup peer 10.10.{{to_id}}.2 {{to_id}}{{id}}21
  backup delay 1 1
!
router ospf 10
 router-id 10.10.{{id}}.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
 !
""")
```

#VSLIDE
### Пример использования Jinja2

Файл routers_info.yml
```yaml
- id: 11
  name: Liverpool
  to_name: LONDON
  IT: 791
  BS: 1550
  to_id: 1

- id: 12
  name: Bristol
  to_name: LONDON
  IT: 793
  BS: 1510
  to_id: 1

- id: 14
  name: Coventry
  to_name: Manchester
  IT: 892
  BS: 1650
  to_id: 2
```

#VSLIDE
### Пример использования Jinja2

Файл router_config_generator.py
```python
# -*- coding: utf-8 -*-
import yaml
from jinja2 import Template
from router_template import template_r1

routers = yaml.load(open('routers_info.yml'))

for router in routers:
    r1_conf = router['name']+'_r1.txt'
    with open(r1_conf,'w') as f:
        f.write(template_r1.render(router))
```

#VSLIDE
### Пример использования Jinja2

Файл router_config_generator.py:
* импортирует шаблон template_r1
* из файла routers_info.yml список параметров считывается в переменную routers

#VSLIDE
### Пример использования Jinja2

Затем в цикле перебираются объекты (словари) в списке routers:
* название файла, в который записывается итоговая конфигурация, состоит из поля name в словаре и строки _r1.txt
 * например, Liverpool_r1.txt
* файл с таким именем открывается в режиме для записи
* в файл записывается результат рендеринга шаблона с использованием текущего словаря
* конструкция with сама закрывает файл
* управление возвращается в начало цикла (пока не переберутся все словари)

#VSLIDE
### Пример использования Jinja2

Запускаем файл router_config_generator.py:
```
$ python router_config_generator.py
```

#VSLIDE
### Пример использования Jinja2

Liverpool_r1.txt:
```
hostname Liverpool
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.11.1 255.255.255.255
!
interface GigabitEthernet0/0
 description WAN to Liverpool sw1 G0/1
!
interface GigabitEthernet0/0.1111
 description MPLS to LONDON
 encapsulation dot1Q 1111
 ip address 10.11.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN Liverpool to sw1 G0/2
!
interface GigabitEthernet0/1.791
 description PW IT Liverpool - LONDON
 encapsulation dot1Q 791
 xconnect 10.10.1.1 1111 encapsulation mpls
  backup peer 10.10.1.2 1121
  backup delay 1 1
!
interface GigabitEthernet0/1.1550
 description PW BS Liverpool - LONDON
 encapsulation dot1Q 1550
 xconnect 10.10.1.1 11111 encapsulation mpls
  backup peer 10.10.1.2 11121
  backup delay 1 1
!
router ospf 10
 router-id 10.10.11.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
!
```



#HSLIDE
## Пример использования Jinja с корректным использованием программного интерфейса

#VSLIDE
### Пример использования Jinja с корректным использованием программного интерфейса

Термин "программный интерфейс" относится к способу работы Jinja с вводными данными и шаблоном, для генерации итоговых файлов. 

Переделанный пример предыдущего скрипта, шаблона и файла с данными (все файлы находятся в каталоге 2_example):

#VSLIDE

Шаблон templates/router_template.txt это обычный текстовый файл:
```
hostname {{name}}
!
interface Loopback10
 description MPLS loopback
 ip address 10.10.{{id}}.1 255.255.255.255
 !
interface GigabitEthernet0/0
 description WAN to {{name}} sw1 G0/1
!
interface GigabitEthernet0/0.1{{id}}1
 description MPLS to {{to_name}}
 encapsulation dot1Q 1{{id}}1
 ip address 10.{{id}}.1.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf hello-interval 1
 ip ospf cost 10
!
interface GigabitEthernet0/1
 description LAN {{name}} to sw1 G0/2 !
interface GigabitEthernet0/1.{{IT}}
 description PW IT {{name}} - {{to_name}}
 encapsulation dot1Q {{IT}}
 xconnect 10.10.{{to_id}}.1 {{id}}11 encapsulation mpls
 backup peer 10.10.{{to_id}}.2 {{id}}21
  backup delay 1 1
!
interface GigabitEthernet0/1.{{BS}}
 description PW BS {{name}} - {{to_name}}
 encapsulation dot1Q {{BS}}
 xconnect 10.10.{{to_id}}.1 {{to_id}}{{id}}11 encapsulation mpls
  backup peer 10.10.{{to_id}}.2 {{to_id}}{{id}}21
  backup delay 1 1
!
router ospf 10
 router-id 10.10.{{id}}.1
 auto-cost reference-bandwidth 10000
 network 10.0.0.0 0.255.255.255 area 0
 !
```

#VSLIDE

Файл с данными routers_info.yml
```
- id: 11
  name: Liverpool
  to_name: LONDON
  IT: 791
  BS: 1550
  to_id: 1

- id: 12
  name: Bristol
  to_name: LONDON
  IT: 793
  BS: 1510
  to_id: 1

- id: 14
  name: Coventry
  to_name: Manchester
  IT: 892
  BS: 1650
  to_id: 2
```

#VSLIDE

Скрипт для генерации конфигураций router_config_generator_ver2.py
```python
# -*- coding: utf-8 -*-
from jinja2 import Environment, FileSystemLoader
import yaml

env = Environment(loader=FileSystemLoader('templates'))
template = env.get_template('router_template.txt')

routers = yaml.load(open('routers_info.yml'))

for router in routers:
    r1_conf = router['name']+'_r1.txt'
    with open(r1_conf,'w') as f:
        f.write(template.render(router))
```

#VSLIDE

Файл router_config_generator.py импортирует из модуля jinja2:
* __FileSystemLoader__ - загрузчик, который позволяет работать с файловой системой
 * тут указывается путь к каталогу, где находятся шаблоны
 * в данном случае, шаблон находится в каталоге templates
* __Environment__ - класс для описания параметров окружения:
 * в данном случае, указан только загрузчик
 * но в нем можно указывать методы обрабатки шаблона

Обратите внимание, что шаблон теперь находится в каталоге __templates__.

#VSLIDE

Если шаблоны находятся в текущем каталоге, надо добавить пару строк и изменить значение в загручике:
```python
import os

curr_dir = os.path.dirname(os.path.abspath(__file__))
env = Environment(loader = FileSystemLoader(curr_dir))
```

Метод __```get_template()```__ используется для того, чтобы получить шаблон. В скобках указывается имя файла.

#HSLIDE
## Синтаксис шаблонов Jinja2

#VSLIDE
## Синтаксис шаблонов Jinja2

До сих пор, в примерах шаблонов Jinja2 использовалась только подстановка переменных.
Это самый простой и понятный пример использования шаблонов.
Но синтаксис шаблонов Jinja на этом не ограничивается.

#VSLIDE
## Синтаксис шаблонов Jinja2

В шаблонах Jinja2 можно использовать:
* переменные
* условия (if/else)
* циклы (for)
* фильтры - специальные встроенные методы, которые позволяют делать преобразования переменных
* тесты - используются для проверки соответствует ли переменная какому-то условию

Кроме того, Jinja поддерживает наследование между шаблонами.
А также позволяет добавлять содержимое одного шаблона в другой.

#VSLIDE
## Синтаксис шаблонов Jinja2

Для генерации шаблонов будет использовать скрипт cfg_gen.py
```python
# -*- coding: utf-8 -*-
from jinja2 import Environment, FileSystemLoader
import yaml
import sys

TEMPLATE_DIR, template = sys.argv[1].split('/')
VARS_FILE = sys.argv[2]

env = Environment(loader=FileSystemLoader(TEMPLATE_DIR),
                  trim_blocks=True, lstrip_blocks=True)
template = env.get_template(template)

vars_dict = yaml.load(open(VARS_FILE))

print(template.render(vars_dict))
```

#VSLIDE
## Синтаксис шаблонов Jinja2

В строке
```
env = Environment(loader = FileSystemLoader(TEMPLATE_DIR), trim_blocks=True)
```

Параметр ```trim_blocks=True``` - удаляет первую пустую строку после блока конструкции, если установлено в True (по умолчанию False).

Также можно добавлять параметр ```lstrip_blocks=True``` - если установлено в True, пробелы и табы в начале строки удаляются (по умолчанию False).

#VSLIDE
## Синтаксис шаблонов Jinja2

Для того, чтобы посмотреть на результат, нужно вызвать скрипт и передать ему два аргумента:
* шаблон
* файл с переменными, в формате YAML

Результат будет выведен на стандартный поток вывода.

#VSLIDE
## Синтаксис шаблонов Jinja2

Пример запуска скрипта:
```
$ python cfg_gen.py templates/variables.txt data_files/vars.yml
```

#HSLIDE
## Контроль символов whitespace

#VSLIDE
### trim_blocks

Параметр ```trim_blocks``` удаляет первую пустую строку после блока конструкции, если его значение равно True (по умолчанию False).

Посмотрим на эффект применения флага на примере шаблона templates/env_flags.txt:
```
router bgp {{ bgp.local_as }}
 {% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}
```

#VSLIDE
### trim_blocks

Если скрипт cfg_gen.py запускается без флагов trim_blocks, lstrip_blocks:
```
env = Environment(loader = FileSystemLoader(TEMPLATE_DIR))
```

Вывод будет таким:
```
$ python cfg_gen.py templates/env_flags.txt data_files/router.yml
router bgp 100

 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100

 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
```

#VSLIDE
### trim_blocks

При добавлении флага trim_blocks таким образом:
```python
env = Environment(loader = FileSystemLoader(TEMPLATE_DIR), trim_blocks=True)
```

Результат выполнения будет таким:
```
$ python cfg_gen.py templates/env_flags.txt data_files/router.yml
router bgp 100
  neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
  neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100

```

#VSLIDE
### lstrip_blocks

Но перед строками ```neighbor ... remote-as``` появились два пробела.
Так получилось из-за того, что перед блоком ```{% for ibgp in bgp.ibgp_neighbors %}``` стоит пробел.
После того, как был отключен лишний перевод строки, пробелы и табы перед блоком добавляются к первой строке блока.

Но это не влияет на следующие строки.
Поэтому строки с ```neighbor ... update-source``` отображаются с одним пробелом.

#VSLIDE
### lstrip_blocks

Параметр ```lstrip_blocks``` контролирует то, будут ли удаляться пробелы и табы от начала строки до начала блока (до открывающейся фигурной скобки).

Если добавить аргумент ```lstrip_blocks=True``` таким образом:
```
env = Environment(loader = FileSystemLoader(TEMPLATE_DIR), trim_blocks=True, lstrip_blocks=True)
```

#VSLIDE
### lstrip_blocks

Результат выполнения будет таким:
```
$ python cfg_gen.py templates/env_flags.txt data_files/router.yml
router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
```

#VSLIDE
### Отключение lstrip_blocks для блока

Иногда, нужно отключить функциональность lstrip_blocks для блока.

Например, если параметр ```lstrip_blocks``` установлен равным True в окружении, но нужно отключить его для второго блока в шаблоне (файл templates/env_flags2.txt):
```
router bgp {{ bgp.local_as }}
 {% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}

router bgp {{ bgp.local_as }}
 {%+ for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}
```

#VSLIDE
### Отключение lstrip_blocks для блока

Результат будет таким:
```
$ python cfg_gen.py templates/env_flags2.txt data_files/router.yml
router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100

router bgp 100
  neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
```

#VSLIDE
### Отключение lstrip_blocks для блока

Плюс после знака процента отключает lstrip_blocks для блока.
В данном случае, только для начала блока.

Если сделать таким образом (плюс добавлен в выражении для завершения блока):
```
router bgp {{ bgp.local_as }}
 {% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}

router bgp {{ bgp.local_as }}
 {%+ for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {%+ endfor %}
```

#VSLIDE
### Отключение lstrip_blocks для блока

Он будет отключен и для конца блока:
```
$ python cfg_gen.py templates/env_flags2.txt data_files/router.yml
router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100

router bgp 100
  neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
  neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
```

#VSLIDE
### Удаление whitespace в блоке

Аналогичным образом можно контролировать удаление whitespace для блока.

Для этого примера в окружении не выставлены флаги:
```
env = Environment(loader = FileSystemLoader(TEMPLATE_DIR))
```

#VSLIDE
### Удаление whitespace в блоке

Шаблон templates/env_flags3.txt:
```
router bgp {{ bgp.local_as }}
 {% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}

router bgp {{ bgp.local_as }}
 {%- for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}
```

Обратите внимание на минус в начале второго блока.
Минут удаляет все whitespace символы. В данном случае, в начале блока.

#VSLIDE
### Удаление whitespace в блоке

Результат будет таким:
```
$ python cfg_gen.py templates/env_flags3.txt data_files/router.yml
router bgp 100

 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100

 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100


router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100

 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100

```

#VSLIDE
### Удаление whitespace в блоке

Если добавить минут в конец блока:
```
router bgp {{ bgp.local_as }}
 {% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {% endfor %}

router bgp {{ bgp.local_as }}
 {%- for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
 {%- endfor %}
```

#VSLIDE
### Удаление whitespace в блоке

Удалится пустая строка и в конце блока:
```
$ python cfg_gen.py templates/env_flags3.txt data_files/router.yml
router bgp 100

 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100

 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100


router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
```

#HSLIDE
## Переменные

#VSLIDE
### Переменные

Переменные в шаблоне указываются в двойных фигурных скобках:
```
hostname {{ name }}

interface Loopback0
 ip address 10.0.0.{{ id }} 255.255.255.255
```

#VSLIDE
### Переменные

Значения переменных подставляются на основе словаря, который передается шаблону.

Переменная, которая передается в словаре, может быть не только числом или строкой, но и, например, списком или словарем.
Внутри шаблона можно, соответственно, обращаться к элементу по номеру или по ключу.

#VSLIDE
### Переменные

Пример шаблона templates/variables.txt, с использованием разных вариантов переменных:
```
hostname {{ name }}

interface Loopback0
 ip address 10.0.0.{{ id }} 255.255.255.255

vlan {{ vlans[0] }}

router ospf 1
 router-id 10.0.0.{{ id }}
 auto-cost reference-bandwidth 10000
 network {{ ospf.network }} area {{ ospf['area'] }}
```

#VSLIDE
### Переменные

И соответствующий файл data_files/vars.yml с переменными:
```
id: 3
name: R3
vlans:
  - 10
  - 20
  - 30
ospf:
  network: 10.0.1.0 0.0.0.255
  area: 0
```

#VSLIDE
### Переменные

Обратите внимание на использование переменной vlans в шаблоне:
* так как переменная vlans это список, можно указывать какой именно элемент из списка нам нужен

Если передается словарь (как в случае с переменной ospf), то внутри шаблона можно обращаться к объектам словаря, используя один из вариантов:
* ospf.network или ospf['network']

#VSLIDE
### Переменные

Результат запуска скрипта будет таким:
```
$ python cfg_gen.py templates/variables.txt data_files/vars.yml
hostname R3

interface Loopback0
 ip address 10.0.0.3 255.255.255.255

vlan 10

router ospf 1
 router-id 10.0.0.3
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
```

#HSLIDE
## Цикл for

#VSLIDE
### Цикл for

Цикл for позволяет проходится по элементам последовательности.

Цикл for должен находится внутри символов ```{% %}```.
Кроме того, нужно явно указывать завершение цикла:
```
{% for vlan in vlans %}
  vlan {{ vlan }}
{% endfor %}
```

#VSLIDE
### Цикл for

Пример шаблона templates/for.txt с использованием цикла:
```
hostname {{ name }}

interface Loopback0
 ip address 10.0.0.{{ id }} 255.255.255.255

{% for vlan, name in vlans.items() %}
vlan {{ vlan }}
 name {{ name }}
{% endfor %}

router ospf 1
 router-id 10.0.0.{{ id }}
 auto-cost reference-bandwidth 10000
 {% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
 {% endfor %}
```

#VSLIDE
### Цикл for

Файл data_files/for.yml с переменными:
```yml
id: 3
name: R3
vlans:
  10: Marketing
  20: Voice
  30: Management
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
### Цикл for

В цикле for можно проходится как по элементам списка (например, список ospf), так и по словарю (словарь vlans). И, аналогичным образом, по любой последовательности.

#VSLIDE
### Цикл for

Результат выполнения будет таким:
```
$ python cfg_gen.py templates/for.txt data_files/for.yml
hostname R3

interface Loopback0
 ip address 10.0.0.3 255.255.255.255

vlan 10
 name Marketing
vlan 20
 name Voice
vlan 30
 name Management

router ospf 1
 router-id 10.0.0.3
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

#HSLIDE

##  if/elif/else

#VSLIDE
###  if/elif/else

if позволяет добавлять условие в шаблон. Например, можно использовать if чтобы добавлять какие-то части шаблона, в зависимости от наличия переменных в словаре с данными.

#VSLIDE
###  if/elif/else

Конструкция if также должна находиться внутри ```{% %}```.
И нужно явно указывать окончание условия:
```
{% if ospf %}
router ospf 1
 router-id 10.0.0.{{ id }}
 auto-cost reference-bandwidth 10000
{% endif %}
```

#VSLIDE
###  if/elif/else

Пример шаблона templates/if.txt:
```
hostname {{ name }}

interface Loopback0
 ip address 10.0.0.{{ id }} 255.255.255.255

{% for vlan, name in vlans.items() %}
vlan {{ vlan }}
 name {{ name }}
{% endfor %}

{% if ospf %}
router ospf 1
 router-id 10.0.0.{{ id }}
 auto-cost reference-bandwidth 10000
 {% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
 {% endfor %}
{% endif %}
```

#VSLIDE
###  if/elif/else

Выражение ```if ospf``` работает так же, как в Python: если переменная существует и не пустая, результат будет True. Если переменной нет, или она пустая, результат будет False.

То есть, в этом шаблоне конфигурация OSPF генерируется только в том случае, если переменная ospf существует и не пустая.

#VSLIDE
###  if/elif/else

Конфигурация будет генерироваться с двумя вариантами данных.

Сначала, с файлом data_files/if.yml, в котором нет переменной ospf:
```yml
id: 3
name: R3
vlans:
  10: Marketing
  20: Voice
  30: Management
```

#VSLIDE
###  if/elif/else

Результат будет таким:
```
$ python cfg_gen.py templates/if.txt data_files/if.yml

hostname R3

interface Loopback0
 ip address 10.0.0.3 255.255.255.255

vlan 10
 name Marketing
vlan 20
 name Voice
vlan 30
 name Management

```

#VSLIDE
###  if/elif/else

Теперь аналогичный шаблон, но с файлом data_files/if_ospf.yml:
```yml
id: 3
name: R3
vlans:
  10: Marketing
  20: Voice
  30: Management
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
###  if/elif/else

Теперь результат выполнения будет таким:
```
hostname R3

interface Loopback0
 ip address 10.0.0.3 255.255.255.255

vlan 10
 name Marketing
vlan 20
 name Voice
vlan 30
 name Management

router ospf 1
 router-id 10.0.0.3
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

#VSLIDE
###  if/elif/else

Как и в Python, в Jinja можно делать ответвления в условии.

Пример шаблона templates/if_vlans.txt:
```
{% for intf, params in trunks.items() %}
interface {{ intf }}
 {% if params.action == 'add' %}
 switchport trunk allowed vlan add {{ params.vlans }}
 {% elif params.action == 'delete' %}
 switchport trunk allowed vlan remove {{ params.vlans }}
 {% else %}
 switchport trunk allowed vlan {{ params.vlans }}
 {% endif %}
{% endfor %}
```

#VSLIDE
###  if/elif/else

Файл data_files/if_vlans.yml с данными:
```yml
trunks:
  Fa0/1:
    action: add
    vlans: 10,20
  Fa0/2:
    action: only
    vlans: 10,30
  Fa0/3:
    action: delete
    vlans: 10
```

В данном примере, в зависимости от значения параметра action, генерируются разные команды.

#VSLIDE
###  if/elif/else

В шаблоне можно было использовать и такой вариант обращения к вложенным словарям:
```
{% for intf in trunks %}
interface {{ intf }}
 {% if trunks[intf]['action'] == 'add' %}
 switchport trunk allowed vlan add {{ trunks[intf]['vlans'] }}
 {% elif trunks[intf]['action'] == 'delete' %}
 switchport trunk allowed vlan remove {{ trunks[intf]['vlans'] }}
 {% else %}
 switchport trunk allowed vlan {{ trunks[intf]['vlans'] }}
 {% endif %}
{% endfor %}
```

#VSLIDE
###  if/elif/else

В итоге, будет сгенерирована такая конфигурация:
```
$ python cfg_gen.py templates/if_vlans.txt data_files/if_vlans.yml
interface Fa0/1
 switchport trunk allowed vlan add 10,20
interface Fa0/3
  switchport trunk allowed vlan remove 10
interface Fa0/2
  switchport trunk allowed vlan 10,30
```

#VSLIDE
###  if/elif/else

Также, с помощью if, можно фильтровать по каким элементам последовательности пройдется цикл for.

Пример шаблона templates/if_for.txt с фильтром, в цикле for:
```
{% for vlan, name in vlans.items() if vlan > 15 %}
vlan {{ vlan }}
 name {{ name }}
{% endfor %}
```

#VSLIDE
###  if/elif/else

Файл с данными (data_files/if_for.yml):
```yml
vlans:
  10: Marketing
  20: Voice
  30: Management
```

#VSLIDE
###  if/elif/else

Результат выполнения:
```
$ python cfg_gen.py templates/if_for.txt data_files/if_for.yml
vlan 20
 name Voice
vlan 30
 name Management
```

#HSLIDE
## Фильтры

#VSLIDE
### Фильтры

В Jinja переменные можно изменять с помощью фильтров.
Фильтры отделяются от переменной вертикальной чертой (pipe ```|```) и могут содержать дополнительные аргументы.

Кроме того, к переменной могут быть применены несколько фильтров.
В таком случае, фильтры просто пишутся последовательно, и каждый из них отделен вертикальной чертой.

#VSLIDE
### Фильтры

Jinja поддерживает большое количество встроенных фильтров.
Мы рассмотрим лишь несколько из них.
Остальные фильтры можно найти в [документации](http://jinja.pocoo.org/docs/dev/templates/#builtin-filters).

Также, достаточно легко, можно создавать и свои собственные фильтры.
Мы не будем рассматривать эту возможность, но это хорошо описано в [документации ](http://jinja.pocoo.org/docs/2.9/api/#custom-filters).

#VSLIDE
### default

Фильтр default позволяет указать для переменной значение по умолчанию.
Если переменная определена, будет выводиться переменная, если переменная не определена, будет выводиться значение, которое указано в фильтре default.

#VSLIDE
### default

Пример шаблона templates/filter_default.txt:
```
router ospf 1
 auto-cost reference-bandwidth {{ ref_bw | default(10000) }}
 {% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
 {% endfor %}
```

Если переменная ref_bw определена в словаре, будет подставлено её значение.
Если же переменной нет, будет подставлено значение 10000.

#VSLIDE
### default

Файл с данными (data_files/filter_default.yml):
```yml
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
### default

Результат выполнения:
```
$ python cfg_gen.py templates/filter_default.txt data_files/filter_default.yml
router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

#VSLIDE
### default

По умолчанию, если переменная определена и её значение пустой объект, будет считаться, что переменная и её значение есть.

Если нужно сделать так, чтобы значение по умолчанию подставлялось и в том случае, когда переменная пустая (то есть, обрабатывается как False в Python), надо указать дополнительный параметр ```boolean=true```.

#VSLIDE
### default

Например, если файл данных был бы таким:
```
ref_bw: ''
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
### default

То в итоге сгенерировался такой результат:
```
$ python cfg_gen.py templates/filter_default.txt data_files/filter_default.yml
router ospf 1
 auto-cost reference-bandwidth 
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

#VSLIDE
### default

Если же, при таком же файле данных, изменить шаблон таким образом:
```
router ospf 1
 auto-cost reference-bandwidth {{ ref_bw | default(10000, boolean=true) }}
{% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
{% endfor %}
```

Вместо ```default(10000, boolean=true)```, можно написать default(10000, true)

#VSLIDE
### default

Результат уже будет таким (значение по умолчанию подставится):
```
$ python cfg_gen.py templates/filter_default.txt data_files/filter_default.yml
router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

#VSLIDE
### dictsort

Фильтр dictsort позволяет сортировать словарь.
По умолчанию, сортировка выполняется по ключам.
Но, изменив параметры фильтра, можно выполнять сортировку по значениям.

Синтаксис фильтра:
```
dictsort(value, case_sensitive=False, by='key')
```

После того, как dictsort отсортировал словарь, он возвращает список кортежей, а не словарь.

#VSLIDE
### dictsort

Пример шаблона templates/filter_dictsort.txt с использованием фильтра dictsort:
```
{% for intf, params in trunks | dictsort %}
interface {{ intf }}
 {% if params.action == 'add' %}
 switchport trunk allowed vlan add {{ params.vlans }}
 {% elif params.action == 'delete' %}
 switchport trunk allowed vlan remove {{ params.vlans }}
 {% else %}
 switchport trunk allowed vlan {{ params.vlans }}
 {% endif %}
{% endfor %}
```

Обратите внимание, что фильтр ожидает словарь, а не список кортежей или итератор.

#VSLIDE
### dictsort

Файл с данными (data_files/filter_dictsort.yml):
```yml
trunks:
  Fa0/1:
    action: add
    vlans: 10,20
  Fa0/2:
    action: only
    vlans: 10,30
  Fa0/3:
    action: delete
    vlans: 10
```

#VSLIDE
### dictsort

Результат выполнения будет таким (интерфейсы упорядочены):
```
$ python cfg_gen.py templates/filter_dictsort.txt data_files/filter_dictsort.yml
interface Fa0/1
  switchport trunk allowed vlan add 10,20
interface Fa0/2
  switchport trunk allowed vlan 10,30
interface Fa0/3
  switchport trunk allowed vlan remove 10
```


#VSLIDE
### join

Фильтр join работает так же, как и метод join в Python.

С помощью фильтра join можно объединять элементы последовательности в строку, с опциональным разделителем между элементами.

#VSLIDE
### join

Пример шаблона templates/filter_join.txt с использованием фильтра join:
```
{% for intf, params in trunks | dictsort %}
interface {{ intf }}
 {% if params.action == 'add' %}
 switchport trunk allowed vlan add {{ params.vlans | join(',') }}
 {% elif params.action == 'delete' %}
 switchport trunk allowed vlan remove {{ params.vlans | join(',') }}
 {% else %}
 switchport trunk allowed vlan {{ params.vlans | join(',') }}
 {% endif %}
{% endfor %}
```

#VSLIDE
### join

Файл с данными (data_files/filter_join.yml):
```yml
trunks:
  Fa0/1:
    action: add
    vlans:
      - 10
      - 20
  Fa0/2:
    action: only
    vlans:
      - 10
      - 30
  Fa0/3:
    action: delete
    vlans:
      - 10
```

#VSLIDE
### join

Результат выполнения:
```
$ python cfg_gen.py templates/filter_join.txt data_files/filter_join.yml
interface Fa0/1
  switchport trunk allowed vlan add 10,20
interface Fa0/2
  switchport trunk allowed vlan 10,30
interface Fa0/3
  switchport trunk allowed vlan remove 10

```

#HSLIDE
## Тесты

#VSLIDE
### Тесты

Кроме фильтров, Jinja также поддерживает тесты.
Тесты позволяют проверять переменные на какое-то условие.

Jinja поддерживает большое количество встроенных тестов.
Мы рассмотрим лишь несколько из них.
Остальные тесты вы можете найти в [документации](http://jinja.pocoo.org/docs/dev/templates/#builtin-tests).

Тесты, как и фильтры, можно создавать самостоятельно.

#VSLIDE
### defined

Тест defined позволяет проверить есть ли переменная в словаре данных.

Пример шаблона templates/test_defined.txt:
```
router ospf 1
{% if ref_bw is defined %}
 auto-cost reference-bandwidth {{ ref_bw }}
{% else %}
 auto-cost reference-bandwidth 10000
{% endif %}
{% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
{% endfor %}
```

#VSLIDE
### defined

Этот пример более громоздкий, чем вариант с использованием фильтра default, но этот тест может быть полезен в том случае, если, в зависимости от того, определена переменная или нет, нужно выполнять разные команды.

Файл с данными (data_files/test_defined.yml):
```yml
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
### defined

Результат выполнения:
```
$ python cfg_gen.py templates/test_defined.txt data_files/test_defined.yml
router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

#VSLIDE
### iterable

Тест iterable проверяет является ли объект итератором.

Благодаря таким проверкам, можно делать ответвления в шаблоне, которые будут учитывать тип переменной.

#VSLIDE
### iterable

Шаблон templates/test_iterable.txt (сделаны отступы, чтобы былы понятней ответвления):
```
{% for intf, params in trunks | dictsort %}
interface {{ intf }}
 {% if params.vlans is iterable %}
   {% if params.action == 'add' %}
 switchport trunk allowed vlan add {{ params.vlans | join(',') }}
   {% elif params.action == 'delete' %}
 switchport trunk allowed vlan remove {{ params.vlans | join(',') }}
   {% else %}
 switchport trunk allowed vlan {{ params.vlans | join(',') }}
   {% endif %}
 {% else %}
   {% if params.action == 'add' %}
 switchport trunk allowed vlan add {{ params.vlans }}
   {% elif params.action == 'delete' %}
 switchport trunk allowed vlan remove {{ params.vlans }}
   {% else %}
 switchport trunk allowed vlan {{ params.vlans }}
   {% endif %}
 {% endif %}
{% endfor %}
```

#VSLIDE
### iterable

Файл с данными (data_files/test_iterable.yml):
```yml
trunks:
  Fa0/1:
    action: add
    vlans:
      - 10
      - 20
  Fa0/2:
    action: only
    vlans:
      - 10
      - 30
  Fa0/3:
    action: delete
    vlans: 10
```

#VSLIDE
### iterable

Обратите внимание на последнюю строку: ```vlans: 10```.
В данном случае, 10 уже не находится в списке и фильтр join в таком случае не работает.
Но, засчет теста ```is iterable``` (в этом случае результат будет false), в этом случае шаблон уходит в ветку else.


#VSLIDE
### iterable

Результат выполнения:
```
$ python cfg_gen.py templates/test_iterable.txt data_files/test_iterable.yml
  interface Fa0/1
            switchport trunk allowed vlan add 10,20
        interface Fa0/2
            switchport trunk allowed vlan 10,30
        interface Fa0/3
            switchport trunk allowed vlan remove 10
```

Такие отступы получились из-за того, что в шаблоне используются отступы, но не установлено lstrip_blocks=True (он удалет пробелы и табы в начале строки).

#HSLIDE
## set

#VSLIDE
### set

Внутри шаблона можно присваивать значения переменным.
Это могут быть новые переменные, а могут быть измененные значения переменных, которые были переданы шаблону.

Таким образом можно запомнить значение, которое, например, было получено в результате применения нескольких фильтров.
И в дальнейшем использовать имя переменной, а не повторять снова все фильтры.

#VSLIDE
### set

Пример шаблона templates/set.txt, в котором выражение set используется чтобы задать более короткие имена параметрам:
```
{% for intf, params in trunks | dictsort %}
 {% set vlans = params.vlans %}
 {% set action = params.action %}

interface {{ intf }}
 {% if vlans is iterable %}
  {% if action == 'add' %}
 switchport trunk allowed vlan add {{ vlans | join(',') }}
  {% elif action == 'delete' %}
 switchport trunk allowed vlan remove {{ vlans | join(',') }}
  {% else %}
 switchport trunk allowed vlan {{ vlans | join(',') }}
  {% endif %}
 {% else %}
  {% if action == 'add' %}
 switchport trunk allowed vlan add {{ vlans }}
  {% elif action == 'delete' %}
 switchport trunk allowed vlan remove {{ vlans }}
  {% else %}
 switchport trunk allowed vlan {{ vlans }}
  {% endif %}
 {% endif %}
{% endfor %}
```

#VSLIDE
### set

Обратите внимание на вторую и третюю строки:
```
  {% set vlans = params.vlans %}
  {% set action = params.action %}
```

Таким образом созданы новые переменные и дальше используются уже эти новые значения.
Так шаблон выглядит понятней.

#VSLIDE
### set

Файл с данными (data_files/set.yml):
```json
trunks:
  Fa0/1:
    action: add
    vlans:
      - 10
      - 20
  Fa0/2:
    action: only
    vlans:
      - 10
      - 30
  Fa0/3:
    action: delete
    vlans: 10
```

#VSLIDE
### set

Результат выполнения:
```
$ python cfg_gen.py templates/set.txt data_files/set.yml

interface Fa0/1
 switchport trunk allowed vlan add 10,20

interface Fa0/2
 switchport trunk allowed vlan 10,30

interface Fa0/3
 switchport trunk allowed vlan remove 10

```

#HSLIDE
## include

#VSLIDE
### include

Выражение include позволяет добавить один шаблон в другой.

Переменные, которые передаются как данные, должны содержать все данные и для основного шаблона, и для того, который добавлен через include.

#VSLIDE
### include

Шаблон templates/vlans.txt:
```
{% for vlan, name in vlans.items() %}
vlan {{ vlan }}
 name {{ name }}
{% endfor %}
```

#VSLIDE
### include

Шаблон templates/ospf.txt:
```
router ospf 1
 auto-cost reference-bandwidth 10000
{% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
{% endfor %}
```

#VSLIDE
### include

Шаблон templates/bgp.txt:
```
router bgp {{ bgp.local_as }}
{% for ibgp in bgp.ibgp_neighbors %}
 neighbor {{ ibgp }} remote-as {{ bgp.local_as }}
 neighbor {{ ibgp }} update-source {{ bgp.loopback }}
{% endfor %}
{% for ebgp in bgp.ebgp_neighbors %}
 neighbor {{ ebgp }} remote-as {{ bgp.ebgp_neighbors[ebgp] }}
{% endfor %}
```

#VSLIDE
### include

Шаблон templates/switch.txt использует созданные шаблоны ospf и vlans:
```
{% include 'vlans.txt' %}

{% include 'ospf.txt' %}
```

#VSLIDE
### include

Файл с данными, для генерации конфигурации (data_files/switch.yml):
```json
vlans:
  10: Marketing
  20: Voice
  30: Management
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
### include

Результат выполнения скрипта:
```
$ python cfg_gen.py templates/switch.txt data_files/switch.yml
vlan 10
 name Marketing
vlan 20
 name Voice
vlan 30
 name Management

router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
```

Итоговая конфигурация получилась такой, как-будто строки из шаблонов ospf.txt и vlans.txt, находились в шаблоне switch.txt.

#VSLIDE
### include


Шаблон templates/router.txt:
```
{% include 'ospf.txt' %}

{% include 'bgp.txt' %}

logging {{ log_server }}
```

В данном случае, кроме include, добавлена ещё одна строка в шаблон, чтобы показать, что выражения include могут идти вперемешку с обычным шаблоном.

#VSLIDE
### include

Файл с данными (data_files/router.yml):
```json
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
bgp:
  local_as: 100
  loopback: lo100
  ibgp_neighbors:
    - 10.0.0.2
    - 10.0.0.3
  ebgp_neighbors:
    90.1.1.1: 500
    80.1.1.1: 600
log_server: 10.1.1.1
```

#VSLIDE
### include

Результат выполнения скрипта будет таким:
```
$ python cfg_gen.py templates/router.txt data_files/router.yml
router ospf 1
 auto-cost reference-bandwidth 10000
 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0

router bgp 100
 neighbor 10.0.0.2 remote-as 100
 neighbor 10.0.0.2 update-source lo100
 neighbor 10.0.0.3 remote-as 100
 neighbor 10.0.0.3 update-source lo100
 neighbor 90.1.1.1 remote-as 500
 neighbor 80.1.1.1 remote-as 600

logging 10.1.1.1
```

#HSLIDE
## Наследование шаблонов

#VSLIDE
### Наследование шаблонов

Наследование шаблонов это очень мощный функционал, который позволяет избежать повторения одного и того же в разных шаблонов.

При использовании наследования различают:
* __базовый шаблон__ - это шаблон, в котором описывается каркас шаблона.
 * в этом шаблоне могут находится любые обычные выражения или текст. Но, кроме того, в этом шаблоне определяются специальные __блоки (block)__. 
* __дочерний шаблон__ - шаблон, который расширяет базовый шаблон, заполняя обозначенные блоки.
 * дочерние шаблоны могут переписывать или дополнять блоки, определенные в базовом шаблоне.

#VSLIDE
### Наследование шаблонов

Пример базового шаблона templates/base_router.txt:
```
!
{% block services %}
service timestamps debug datetime msec localtime show-timezone year
service timestamps log datetime msec localtime show-timezone year
service password-encryption
service sequence-numbers
{% endblock %}
!
no ip domain lookup
!
ip ssh version 2
!
{% block ospf %}
router ospf 1
 auto-cost reference-bandwidth 10000
{% endblock %}
!
{% block bgp %}
{% endblock %}
!
{% block alias %}
{% endblock %}
!
line con 0
 logging synchronous
 history size 100
line vty 0 4
 logging synchronous
 history size 100
 transport input ssh
!

```

#VSLIDE
### Наследование шаблонов

Обратите внимание на четыре блока, которые созданы в шаблоне:
```
{% block services %}
service timestamps debug datetime msec localtime show-timezone year
service timestamps log datetime msec localtime show-timezone year
service password-encryption
service sequence-numbers
{% endblock %}
!
{% block ospf %}
router ospf 1
 auto-cost reference-bandwidth 10000
{% endblock %}
!
{% block bgp %}
{% endblock %}
!
{% block alias %}
{% endblock %}
```

Это заготовки для соответствующих разделов конфигурации.
Дочерний шаблон, который будет использовать этот базовый шаблон как основу, может заполнять все блоки или только какие-то из них.

#VSLIDE
### Наследование шаблонов

Дочерний шаблон templates/hq_router.txt:
```
{% extends "base_router.txt" %}

{% block ospf %}
{{ super() }}
{% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
{% endfor %}
{% endblock %}

{% block alias %}
alias configure sh do sh
alias exec ospf sh run | s ^router ospf
alias exec bri show ip int bri | exc unass
alias exec id show int desc
alias exec top sh proc cpu sorted | excl 0.00%__0.00%__0.00%
alias exec c conf t
alias exec diff sh archive config differences nvram:startup-config system:running-config
alias exec desc sh int desc | ex down
{% endblock %}
```

#VSLIDE
### Наследование шаблонов

Первая строка в шаблоне templates/hq_router.txt очень важна:
```
{% extends "base_router.txt" %}
```

Именно она говорит о том, что шаблон hq_router.txt будет построен на основе шаблона base_router.txt.

#VSLIDE
### Наследование шаблонов

Внутри дочернего шаблона всё происходит внутри блоков.
Засчет блоков, которые были определены в базовом шаблоне, дочерний шаблон может расширять родительский шаблон.

Обратите внимание, что те строки, которые описаны в дочернем шаблоне за пределами блоков, игнорируются.

#VSLIDE
### Наследование шаблонов

В базовом шаблоне три блока: ospf, bgp, alias.
В дочернем шаблоне заполнены только два из них: ospf и alias.

В этом удобство наследования. Не обязательно заполнять все блоки в каждом дочернем шаблоне.

#VSLIDE
### Наследование шаблонов

При этом, блоки ospf и alias используются по-разному.
В базовом шаблоне, в блоке ospf уже была часть конфигурации:
```
{% block ospf %}
router ospf 1
 auto-cost reference-bandwidth 10000
{% endblock %}
```

#VSLIDE
### Наследование шаблонов

Поэтому, в дочернем шаблоне есть выбор: использовать эту конфигурацию и дополнить её, или полностью переписать всё в дочернем шаблоне.

В данном случае, конфигурация дополняется.

#VSLIDE
### Наследование шаблонов

Именно поэтому в дочернем шаблоне templates/hq_router.txt блок ospf начинается с выражения ```{{ super() }}```:
```
{% block ospf %}
{{ super() }}
 {% for networks in ospf %}
 network {{ networks.network }} area {{ networks.area }}
 {% endfor %}
{% endblock %}
```

#VSLIDE
### Наследование шаблонов

{{ super() }} переносит в дочерний шаблон содержимое этого блока из родительского шаблона.
Засчет этого, в дочерний шаблон перенесутся строки из родительского.

Выражение super не обязательно должно находится в самом начале блока. Оно может быть в любом месте блока. Содержимое базового шаблона, перенесется в то место, где находится выражение super.

#VSLIDE
### Наследование шаблонов

В блоке alias просто описаны нужные alias.
И, даже если бы в родительском шаблоне были какие-то настроки, они были бы затерты содержимым дочернего шаблона.

Файл с данными для генерации конфигурации по шаблону (data_files/hq_router.yml):
```json
ospf:
  - network: 10.0.1.0 0.0.0.255
    area: 0
  - network: 10.0.2.0 0.0.0.255
    area: 2
  - network: 10.1.1.0 0.0.0.255
    area: 0
```

#VSLIDE
### Наследование шаблонов

Результат выполнения будет таким:
```
$ python cfg_gen.py templates/hq_router.txt data_files/hq_router.yml
!
service timestamps debug datetime msec localtime show-timezone year
service timestamps log datetime msec localtime show-timezone year
service password-encryption
service sequence-numbers
!
no ip domain lookup
!
ip ssh version 2
!
router ospf 1
 auto-cost reference-bandwidth 10000

 network 10.0.1.0 0.0.0.255 area 0
 network 10.0.2.0 0.0.0.255 area 2
 network 10.1.1.0 0.0.0.255 area 0
!
!
alias configure sh do sh
alias exec ospf sh run | s ^router ospf
alias exec bri show ip int bri | exc unass
alias exec id show int desc
alias exec top sh proc cpu sorted | excl 0.00%__0.00%__0.00%
alias exec c conf t
alias exec diff sh archive config differences nvram:startup-config system:running-config
alias exec desc sh int desc | ex down
!
line con 0
 logging synchronous
 history size 100
line vty 0 4
 logging synchronous
 history size 100
 transport input ssh
!


```

Как запустить ноду Mineplex и превратить ее в Pool
==================================================

В этом гайде мы пошагово установим и запустим ноду Mineplex, синхронизируем все блоки и запустим процессы по выпечке/подтверждения блоков
Инструкция написана на основе сервера Digital Ocean (Ubuntu 20.04 (LTC) x64)

Подготовка
==========

Обновление пакетов
~~~~~~~~~~~~~~~~~~

::

   Apt-get update
   Apt-get upgrade
   Apt-get dist-upgrade

Перезапуск
~~~~~~~~~~

::

   Reboot

Установка
=========

Установка модулей
~~~~~~~~~~~~~~~~~

::

   sudo apt install -y rsync git m4 build-essential patch unzip wget pkg-config libgmp-dev libev-dev libhidapi-dev libffi-dev opam jq

Убедитесь, что установлены рабочие версии Opam и OCaml
::

   opam --version
   ocaml --version

Рабочей связкой является Opam 2.0.5 и OCaml 4.08.1

Cоздание нового пользователя
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   sudo adduser mineplex

Переходим в нового пользователя, дальнейшие работы будут проводиться через него
::

   su mineplex

Получение исходного кода
~~~~~~~~~~~~~~~~~~~~~~~~

::

   cd ~
   git clone https://github.com/mineplexio/Plexus-Pool.git -b mineplex-beta-protocol mineplex.blockchain

Переходим в директорию mineplex.blockchain
::

   cd mineplex.blockchain

Установка rustup
~~~~~~~~~~~~~~~~

::

   opam init --bare
    
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- --default-toolchain none -y
   source $HOME/.cargo/env
   rustup set profile minimal
   rustup toolchain install 1.39.0
   rustup default 1.39.0
   source $HOME/.cargo/env

Установка компилятора OCaml и зависимостей Mineplex
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::
   
   make build-deps

После установки всего необходимого, мы можем скомпилировать проект
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   eval $(opam env)
   mkdir src/proto_001_Pt8PXNHh/parameters
   make

Запуск ноды
===========

Идентификация ноды
~~~~~~~~~~~~~~~~~~

Первым делом нам необходимо сгенерировать идентификатор, чтобы нода смогла подключиться к сети
::

    ./mineplex-node config init --data-dir ~/mineplex-mainnet
    ./mineplex-node identity generate --data-dir ~/mineplex-mainnet

Запуск ноды и синхронизация блоков
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-node run  --data-dir ~/mineplex-mainnet --rpc-addr 127.0.0.1:8732 --connections 15 --history-mode=archive


Не завершайте данный процесс, держите данную команду запущенной на протяжений всего времени. Завершить этот процесс = выключить ноду

Процесс синхронизации занимает большое количество времени. Поэтому рекомендую запустить данный процесс в фоне (nohup или screen, кому как). Либо вам придется держать сессию открытой (1-2 дня). Есть два способа благодаря которым вы поймете синхронизирована ли ваша нода:

- На старте ваша нода будет затрачивать по ~1 секунде на каждый блок. Если вы заметили, что на каждый блок уходит по одной минуте, значит ваша нода синхронизирована. Т.к. внутри блокчейна на каждый блок уходит по 1 минуте, ваша нода дожидается новых блоков
- Используйте команду ``./mineplex-client -endpoint http://127.0.0.1:8732/ bootstrapped`` При успешной синхронизации команда вернет ``Node is bootstrapped.``

Создание кошелька
~~~~~~~~~~~~~~~~~

::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ gen keys bob

Bob - имя вашего кошелька внутри ноды. Вы можете использовать любое другое, которое вы хотите

После создания кошелька проверьте командой:
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ list known contracts

Поднятие Pool
=============

Раздел для тех, кто хочет стать активном пулом сети. Вы сможете создавать блоки, подтверждать и получать за это награды

Для работы Pool, на вашем балансе должно лежать минимум 2.000.000 Mine. Данные Mine останутся у вас, они служат в качестве депозита. За каждое создание блока берется депозит в 6000 Mine, за подтверждение 200 Mine. После разморозки добытых Plex депозит вернется к вам.

В сети распределение задач происходит случайным образом исходя из вашего ролла. 1 ролл = 1.000.000 Mine. Для примера, если на вашем балансе лежит 2.700.000 Mine, ваш ролл будет равен 2.

Посмотреть данные вашего кошелька
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::
   
   ./mineplex-client show address bob -S

Вывод:
::

   Hash: mp1........ <-- Адрес кошелька
   Public Key: edpk.........
   Secret Key: edsk......... <-- Секретный ключ вашего кошелька

После того, как на балансе вашего пула будет 2.000.000 Mine, можно продолжать

Регистрация и получение прав
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ register key bob as delegate

Проверка
~~~~~~~~

::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ rpc get /chains/main/blocks/head/context/delegates/адрес пула

Вывод:

::

   { "balance": "2099999997579", "frozen_balance": "6800000000",
     "frozen_balance_by_cycle": [], "staking_balance": "2099999997579",
     "delegated_contracts": [ "mp1CnuAo6ENAuaduenfnsULxdaqtWqcsy3cY" ],
     "delegated_balance": "0", "deactivated": false, "grace_period": 347 }

Если всё хорошо, grace_period будет равен текущий цикл + 11

Эта команда для просмотра статуса вашего пула, в будущем здесь будет отображаться информация (Сколько добыто за цикл, общий/пользовательский стейк, делегированные адреса, стасус: выключен/включен пул)

Вам необходимо подождать 7 циклов, после запуска, после этого времени вы начнете создавать/подтверждать/контролировать двойную выпечку.

А пока, вы можете запустить все процессы. (Все процессы нужно запускать в фоне)

Запуск Baker (Создание блоков)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-baker-002-Pt4xzupC run with local node ~/mineplex-mainnet bob

Запуск Endorser (Подтверждение блоков)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-endorser-002-Pt4xzupC run

Запуск Accuser (Обвинитель)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Это процесс, который проверяет все блоки в сети. И ищет Pool который создает двойные блоки и подтверждает несколько раз один и тот же блок на одном слоте. Если он найдет такой Pool, нарушитель потеряет весь свой депозит. Поэтому, внимательно контролируйте запущенные процессы. На одном Pool должен быть запущен только один Baker и только один Endorser.
::

   ./mineplex-accuser-002-Pt4xzupC run

Запуск скрипта для выплат
=========================

Перед стартом необходимо установить некоторые модули

Выходим в пользователя Root
::

   exit

Установка npm
~~~~~~~~~~~~~
::

   apt install npm

Установка MongoDB
~~~~~~~~~~~~~~~~~

Установка MongoDB отличается (в зависимости от вашей ОС и ее версии), рекомендую устанавливать по данной инструкции: https://docs.mongodb.com/manual/administration/install-community/

Получение исходного кода скриптов
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Переходим обратно в пользователя mineplex
::

   su mineplex
   cd ~

Скачиваем скрипты
::

   git clone https://github.com/mineplexio/Pool-Script.git
   cd Pool-Script
   git submodule add https://github.com/mineplexio/js-rpcapi.git
   git submodule update --init  
   git submodule update --remote
   cd js-rpcapi; npm install; cd

Настройка скриптов
~~~~~~~~~~~~~~~~~~
::

   cd Pool-Script
   cp config-example.js config.js
   nano config.js

Перед вами появятся основные настройки по выплатам:
::

   module.exports = {
  "NODE_RPC": "http://127.0.0.1:8732/",
  "MONGO_URL": "mongodb://localhost:27017/dbname",
  "START_INDEXING_LEVEL": 350160, <--Просматривает все циклы начиная с указанного блока на выплаты, можно указать последний блок за прошлый цикл
  "BAKER_LIST": [
    "address" <-- Вставьте адрес вашего Pool
  ],
  "PAYMENT_SCRIPT": {
    "ENABLED_AUTOPAYMENT": true, // Автоматически ежедневно выплачивает Plex.
    "AUTOPAYMENT_LEVEL": 10, // Блок внутри цикла на котором будут происходить выплаты. Минимум - 5, максимум - 1440
    "BAKER_PRIVATE_KEYS": [
      "privatekey" <-- Вставьте секретный ключ вашего Pool
    ],
    "MIN_PAYMENT_AMOUNT": 0.1, // Минимальная награда в PLEX
    "DEFAULT_BAKER_COMMISSION": 0.1, // Комиссия которую берет себе пул за создание блоков 1 = 100%, 0.1 = 10%
    "BAKERS_COMMISSIONS": {
      "address1" : 0.15,
      "address2" : 0.1,
    },
    "ADDRESSES_COMMISSIONS": { // Вы можете поставить разную комиссию на каждый адрес 
      "address3" : 0,
    },
    "MAX_COUNT_OPERATIONS_IN_ONE_BLOCK": 199
  }

Запуск скриптов
~~~~~~~~~~~~~~~

Запускайте в фоне
::

   npm run start


Различные команды
=================

Баланс Mine
~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ get mine_balance for bob

Баланс Plex
~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ get balance for bob

Перевести Mine
~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ mine_transfer 100 from bob to адрес

Перевести Plex
~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ transfer 100 from bob to адрес

Запланированные выпечки
~~~~~~~~~~~~~~~~~~~~~~~
::
   ./mineplex-client -endpoint http://127.0.0.1:8732/ rpc get /chains/main/blocks/head/helpers/baking_rights\?cycle=номер цикла\&delegate=адрес пула\&max_priority=2

Запланированные подтверждения
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::
   ./mineplex-client -endpoint http://127.0.0.1:8732/ rpc get /chains/main/blocks/head/helpers/endorse_rights\?cycle=номер цикла\&delegate=адрес пула\&max_priority=2

Статус пула
~~~~~~~~~~~ 
(Сколько добыто за цикл, общий/пользовательский стейк, делегированные адреса, стасус: выключен/включен пул)
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ rpc get /chains/main/blocks/head/context/delegates/адрес пула

Просмотр режима работы пула
~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client rpc get /chains/main/checkpoint

Список кошельков внутри ноды
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::
   
   ./mineplex-client -endpoint http://127.0.0.1:8732/ list known contracts

Просмотреть информацию по кошельку внутри ноды
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client show address bob -S       префикс -S выводит секретный ключ кошелька

Импортировать адрес в ноду
~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client import address user1 mp1......

Импортировавь секретный ключ кошелька
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client import secret key user1 unencrypted:edsk......

Удалить кошелек из ноды
~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client forget address user1

Делегировать кошелек в пул
~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ set delegate for user1 to bob

Разделегировать кошелек
~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ withdraw delegate from user1

Проверка синхронизации ноды
~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ bootstrapped

Временные метки последнего блока
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ get timestamp

Сгенерировать кошелек (seed фразы не будет)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ gen keys user2

Важные переменные блокчейна
~~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ rpc get /chains/main/blocks/head/context/constants | jq

Просмотреть последний блок
~~~~~~~~~~~~~~~~~~~~~~~~~~
::

   ./mineplex-client -endpoint http://127.0.0.1:8732/ rpc get /chains/main/blocks/head/operations
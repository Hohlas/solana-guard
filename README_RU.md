[English](README.md)
# [guard.sh](https://github.com/Hohlas/solana-guard/blob/main/guard.sh)
Скрипт бесшовного переключения голосования ноды соланы между основным и резервным серверами 

![image](https://github.com/user-attachments/assets/fc4895a3-e356-43f4-b807-dd3a020a8925)

## Основные функции
- Автоматическое переключение голосования на резервный сервер (Secondary) при делинке ноды на основном сервере (Primary).
- Принудительное переключение голосования командой с аргументом 'guard p' при запуске на резервном сервере. Предварительно проверяется запас времени до следующего блока и состояние (health, behind) резервного сервера.
- Циклическая работа - после автоматического переключения голосования перезапуск скрипта не требуется. Статус сервера Primary/Secondary меняетя в зависимости от режима голосования ноды.
- Взаимная проверка функционирования скриптов. Primary сервер контролирует работу скрипта на Secondary сервере, и наоборот, предотвращая случайное отключение мониторинга.
- Мониторинг состояния нод соланы на обоих серверах: next_slot_minutes, health, behind текущего сервера, behind удаленного сервера.

  ![ok_1](https://github.com/user-attachments/assets/86e854f3-dd91-467f-983d-66c5a9fc346a)
  
- Запись всех событий в лог. Фиксируются все отставания и переключения ноды.

  ![log2](https://github.com/user-attachments/assets/f7b1e38e-a728-4728-8256-bb9cc657feb3)
  
- Алерты в телегу о переключениях и отставаниях на обоих серверах.

  ![tg_vote_off](https://github.com/user-attachments/assets/7aa14095-c7bf-48c2-bbe8-8d0accf653e7)
  
- Назначение приоритетного сервера для голосования запуском с аргументом 'guard p'. Голосование постоянно переключается обратно на приоритетный сервер после возвращения его состояния в норму (health, behind).

  ![guard_p](https://github.com/user-attachments/assets/c088160d-1385-48fd-8512-b59d225cee68)

- Переключение голосования при отставании основного сервера на протяжении X*5 секунд, не дожидаясь делинка ноды. Задается аргументом 'guard X'.
  
  ![guard_13](https://github.com/user-attachments/assets/be7acc25-26f4-40a0-84c8-4be7855ece3b)
  
- Перезапуск сервиса соланы в режиме "No Voting" на основном сервере при пропадании на нем интернета.
- Переключение сервиса 'telegraf' на обоих серверах в соответствии с их статусом Primaty/Secondary.
- Переключение сервиса 'relayer' в соответствии со статусом сервера Primaty/Secondary.

## Алгоритм работы резервного сервера Secondary
- Мониторинг состояния резервной и голосующей нод: 'health', 'behind'.
- Сравнивается IP адрес голосующей ноды с локальным IP адресом. Если они совпадают, сервер переходит в статус Primary.
- Переключение голосования с основного на резервный сервер происходит при отсутствии отставания резервной ноды и наступлении одного из трех событий:
	- Статус "Delinquent" голосующей ноды
	- Превышение 'behind time' голосующей ноды порогового значения X*5 секунд, если guard запущен с аргументом 'guard X'.
	- Наличие флага "permanent_primary", если guard запущен с аргументом 'guard p'.
- Переключение голосования с основного на резервный сервер производится в следующей последовательности:
	- Отключение голосования на основном сервере = смена ключа валидатора на неголосующий.
   	- Копирование тауэра с основного сервера. Тауэр копируется каждые 5 секунд на случай отключения интернета на осносном сервере.
   	- Включение голосования на резервном сервере
   	- отключение  сервиса 'telegraf' на основном и запуск на резервном.
   	- отключение  сервиса 'relayer' на основном и запуск на резервном.
   	- Ожидание смены IP адреса ноды в сети для изменения статуса Secondary->Primary.
- Обработка нештатных ситуаций.  
	- При неудачной попытке замены ключа на основном сервере, на нем производится перезапуск сервиса solana в режиме "NoVoting".
	- Если перезапуск сервиса solana так же не удался, резервный сервер пытается перезагрузить основной сервер, отправкой команды 'reboot' по SSH.
 	- Если команда перезагрузки тоже не прошла, проверяется пинг до основного сервера.
 	- Если основной сервер не пингуется, делется вывод о том, что на нем отсутствует связь, и следовательно он самостоятельно перезапустит сервис соланы в режиме "NoVoting".
	- Если основной сервер пингуется, то попытки отключения на нем голосования продолжаются. 
В любом случае для предотвращения ситуации дублирования, перед включением голосования резервный сервер проверяет статус голосования основного командой 
solana-validator --ledger '$LEDGER' contact-info 
и включает голосование лишь в случае, если голосующий ключ не равен идентити ноды.
 
## Алгоритм работы основного сервера Primary
- При пропадании интернет соединения более 15 секунд происходит перезапуск сервиса соланы в режиме "No Voting".
- Сравнивается IP адрес голосующей ноды с локальным IP адресом. Если они не равны, сервер переходит в статус Secondary.
- Мониторинг состояния голосующей и резервной нод: 'health', 'behind'. 

## Установка guard.sh
Загрузка последней версии guard.sh и добавление алиаса
```bash
# install curl jq bc
if ! command -v curl >/dev/null || ! command -v jq >/dev/null || ! command -v bc >/dev/null; then
    echo "install curl jq bc..."
    sudo apt update && sudo apt install -y curl jq bc
fi

# download guard.sh
LATEST_TAG_URL=https://api.github.com/repos/Hohlas/solana-guard/releases/latest
TAG=$(curl -sSL "$LATEST_TAG_URL" | jq -r '.tag_name')

if [ -d ~/solana-guard ]; then 
	echo "update latest release $TAG"
	curl -sSL https://raw.githubusercontent.com/Hohlas/solana-guard/$TAG/guard.sh > $HOME/solana-guard/guard.sh
	curl -sSL https://raw.githubusercontent.com/Hohlas/solana-guard/$TAG/check.sh > $HOME/solana-guard/check.sh
else 
	echo "clone latest release $TAG"
	git clone https://github.com/Hohlas/solana-guard.git ~/solana-guard
	cd ~/solana-guard
	git fetch --tags 
	git checkout tags/$TAG
	git submodule update --init --recursive
fi

chmod +x ~/solana-guard/guard.sh
chmod +x ~/solana-guard/check.sh

# set aliases
BASHRC_FILE="$HOME/.bashrc"
if [ -f "$HOME/.bashrc" ]; then
	NEW_ALIAS="alias guard='source ~/solana-guard/guard.sh'"
	if ! grep -qxF "$NEW_ALIAS" "$BASHRC_FILE"; then
		echo "$NEW_ALIAS" >> "$BASHRC_FILE"
		echo "add new alias: [$NEW_ALIAS]"
	fi
	NEW_ALIAS="alias check='source ~/solana-guard/check.sh'"
	if ! grep -qxF "$NEW_ALIAS" "$BASHRC_FILE"; then
		echo "$NEW_ALIAS" >> "$BASHRC_FILE"
		echo "add new alias: [$NEW_ALIAS]"
	fi
else
    echo "file $HOME/.bashrc not found."
fi

source $HOME/.bashrc
```

Сервис соланы всегда должен запускаться с неголосующим ключем 'empty-validator.json'.
Генерация неголосующего ключа
```bash
if [ ! -f ~/solana/empty-validator.json ]; then 
solana-keygen new -s --no-bip39-passphrase -o ~/solana/empty-validator.json
fi
```
Изменение solana.service
```bash
--identity /root/solana/empty-validator.json \
--authorized-voter /root/solana/validator-keypair.json \
--vote-account /root/solana/vote.json \
```
Создание папки ~/keys на рамдиске
```bash
if [ ! -d "/mnt/keys" ]; then
    mkdir -p /mnt/keys
    chmod 600 /mnt/keys 
	echo "# KEYS to RAMDISK 
	tmpfs /mnt/keys tmpfs nodev,nosuid,noexec,nodiratime,size=1M 0 0" | sudo tee -a /etc/fstab
	mount /mnt/keys
	echo "create and mount /mnt/keys in RAMDISK"
else
    echo "/mnt/keys exist"
fi
```
Создание символических ссылок на папку с ключами /mnt/keys
```bash
# create links
ln -sf /mnt/keys ~/keys
ln -sf /mnt/keys/vote-keypair.json ~/solana/vote.json
ln -sf /mnt/keys/validator-keypair.json ~/solana/validator-keypair.json
```
Настройки находятся в файле ~/guard.cfg
Для отправки оповещений через телеграмм потребуется BOT_TOKEN.
Поскольку РПЦ соланы иногда возвращает неверные значения IP адреса голосующей ноды добавлен альтернативный [РПЦ Хелиуса](https://dashboard.helius.dev).
Делается запрос на каждый из серверов: соланы и хелиуса. Если они совпадают, ответ принимается. Если не совпадают, делается дополнительно по 10 запросов на каждый из РПЦ.
Ответ принимается, преобладает 70% ответов.  
Одного бесплатного аккаунта Хелиуса хватает на три дня, поэтому сервера берутся из списка RPC_LIST поочередно. 
Как только один сервер перестает отвечать, скрипт опрашивает слудующий по списку.
```bash
PORT='22' # remote server ssh port
KEYS=$HOME/keys
SOLANA_SERVICE=$HOME/solana/solana.service
BEHIND_WARNING=false # 'false'- send telegramm INFO missage, when behind. 'true'-send ALERT message
WARNING_FREQUENCY=12 # max frequency of warning messages (WARNING_FREQUENCY x 5) seconds
BEHIND_OK_VAL=3 # behind, that seemed ordinary
RELAYER_SERVICE=false # use restarting jito-relayer service
CHAT_ALARM=-100..684
CHAT_INFO=-100..888
BOT_TOKEN=507625...VICllWU
RPC_LIST=(
"https://mainnet.helius-rpc.com/..."
"https://mainnet.helius-rpc.com/..."
"https://mainnet.helius-rpc.com/..."
)
```
Ключи private_key.ssh от обоих серверов должны находиться в папках ~/keys.
В первый раз скрипт резервного сервера должен запускаться перед запуском на основном сервере. 
При этом на основной сервер копируется файл с IP адресом резервного. Даллее порядок запуска не имеет значения.  

## check.sh
Удобное отслеживание статуса сервера  

![image](https://github.com/user-attachments/assets/efe5076e-ead2-4ec9-841a-e9ed61a4d309)



**Задание 1**

* Составьте команду rsync, которая позволяет создавать зеркальную копию домашней директории пользователя в директорию /tmp/backup  
* Необходимо исключить из синхронизации все директории, начинающиеся с точки (скрытые)  
* Необходимо сделать так, чтобы rsync подсчитывал хэш-суммы для всех файлов, даже если их время модификации и размер идентичны в источнике и приемнике.  
* На проверку направить скриншот с командой и результатом ее выполнения

rsync \-a \--progress . /tmp/backup  
![][image1]

rsync \-ac \--exclude '.\*' \--delete \--progress . /tmp/backup  
![][image2]  
![][image3]

 rsync \-ac \--delete \--delete-excluded  \--exclude '.\*' \--progress . /tmp/backup  
\--delete-excluded — удаляет из приемника скрытые файлы и папки (.\*), если они успели скопироваться туда во время прошлых запусков.

**Задание 2**

* Написать скрипт и настроить задачу на регулярное резервное копирование домашней директории пользователя с помощью rsync и cron.  
* Резервная копия должна быть полностью зеркальной  
* Резервная копия должна создаваться раз в день, в системном логе должна появляться запись об успешном или неуспешном выполнении операции  
* Резервная копия размещается локально, в директории /tmp/backup  
* На проверку направить файл crontab и скриншот с результатом работы утилиты.

Скрипт:

| \#\!/bin/bash SOURCE="\$HOME/" TARGET="/tmp/backup" mkdir \-p "\$TARGET" rsync \-ac \--exclude ".\*" \--delete "\$SOURCE" "\$TARGET" 2\> "\$HOME/backup\_errors.log" \#Проверка выполнения if \[ \$? \-eq 0 \]; then     echo "\$(date '+%Y-%m-%d %H:%M:%S') \- Бэкап успешно завершен" \>\> "\$HOME/backup.log" else     echo "\$(date '+%Y-%m-%d %H:%M:%S') \- Ошибка при выполнении бэкапа\! Проверьте backup\_errors.log" \>\> "\$HOME/backup.lo\> fi |
| :---- |

![][image4]  
sudo chmod \+x [backup.sh](http://backup.sh)

crontab \-e \# Каждый день в 5 утра  
![][image5]  
![][image6]  
**Задание 3\***

* Настройте ограничение на используемую пропускную способность rsync до 1 Мбит/c  
* Проверьте настройку, синхронизируя большой файл между двумя серверами  
* На проверку направьте команду и результат ее выполнения в виде скриншота

Создадим файл для копирования:  
fallocate \-l 1G largefile

Утилита принимает значение по умолчанию в килобайтах в секунду (КБ/с). Так как 1 Мбит/с равен примерно 125 КБ/с (1000 Кбит / 8), нужно указать значение 125 

rsync \-ac \--bwlimit=125 \--delete –progress largefile siadmin@192.168.0.12:/home/siadmin  
![][image7]  
![][image8]  
**Задание 4\***

* Напишите скрипт, который будет производить инкрементное резервное копирование домашней директории пользователя с помощью rsync на другой сервер  
* Скрипт должен удалять старые резервные копии (сохранять только последние 5 штук)  
* Напишите скрипт управления резервными копиями, в нем можно выбрать резервную копию и данные восстановятся к состоянию на момент создания данной резервной копии.  
* На проверку направьте скрипт и скриншоты, демонстрирующие его работу в различных сценариях.

| rsync \-ac \--delete \--delete-excluded  \--exclude '.\*' \--progress . siadmin@192.168.0.12:/home/siadmin/BACKUP |
| :---- |

![][image9]  
![][image10]

Создадим папки вида 2026-10-23\_06-46-17   
находим последнюю:  
find . \-maxdepth 1 \-type d \-name '20\[0-9\]\[0-9\]-\*' \-printf '%P\\n' 2\>/dev/null \| sort \| tail \-n 1

| \#\!/bin/bash SRC\_DIR="\$HOME" REMOTE\_USER="siadmin" REMOTE\_HOST="192.168.0.12" REMOTE\_PORT="22" REMOTE\_BACKUP\_DIR="/home/siadmin/BACKUP" \# Текущая дата и время для новой папки DATE\_TAG=\$(date \+%Y-%m-%d\_%H-%M-%S) CURRENT\_BACKUP\_PATH="\${REMOTE\_BACKUP\_DIR}/\${DATE\_TAG}" \#echo \$CURRENT\_BACKUP\_PATH \#ssh \-p \$REMOTE\_PORT \$REMOTE\_USER@\$REMOTE\_HOST "mkdir \-p \${REMOTE\_BACKUP\_DIR}" \# Поиск последней созданной копии для жестких ссылок (--link-dest) LATEST\_BACKUP=\$(ssh \-p \$REMOTE\_PORT \$REMOTE\_USER@\$REMOTE\_HOST "find \${REMOTE\_BACKUP\_DIR} \-maxdepth 1 \-type d \-name '20\[0-9\]\[0-9\]-\*' \-printf '%P\\n' 2\>/dev/null \| sort \| tail \-n 1") echo \$LATEST\_BACKUP LINK\_DEST\_PARAM="" \#Если найден предыдущий бекап то if \[ \-n "\$LATEST\_BACKUP" \]; then     LINK\_DEST\_PARAM="--link-dest=../\${LATEST\_BACKUP}" fi \#иначе создается полная копия \# Выполнение rsync rsync \-avz \--delete \\     \$LINK\_DEST\_PARAM \\     \--exclude '.cache/' \\     \--exclude 'Downloads/' \\     \--exclude '.virtualenvs/' \\     \--exclude ".\*" \\     \-e "ssh \-p \$REMOTE\_PORT" \\     "\$SRC\_DIR/" "\$REMOTE\_USER@\$REMOTE\_HOST:\$CURRENT\_BACKUP\_PATH" if \[ \$? \-eq 0 \]; then         echo "Резервная копия успешно создана в: \$CURRENT\_BACKUP\_PATH" else         echo "ERROR BACKUP" fi |
| :---- |

![][image11]  
![][image12]  
\*скрипт запрашивает пароль дважды, потому что внутри него выполняются две независимые сетевые команды: сначала проверка наличия прошлых бэкапов через ssh, а затем сама передача файлов через rsync 

Создадим ключ:  
Проверка существует ли ls \-l \~/.ssh/id\_rsa.pub   
ssh-keygen \-t rsa \-b 4000   
ssh-copy-id \-p 22 siadmin@192.168.0.12

Теперь скрипт работает без ввода пароля   
![][image13]
# Wordpress #

Конструктор сайтов wordpress + mysgl + nginx + certbot (letsencrypt).
Собрано по статье https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-docker-compose-ru

## Образы

- mysql:8.0
- wordpress:5.1.1-fpm-alpine
- nginx:1.15.12-alpine
- certbot/certbot

## Переменные
Замените переменные в файле .env на необходимые значения (адреса доменов, учетные данные mysql, адрес почты)

## Сертификат

При первом запуске Certbot выполнит запрос сертификата letsencrypt с параметрами из .env. 
Запуск Certbot происходит с командой:
`command: certonly --webroot --webroot-path=/var/www/html --email $MYEMAIL --agree-tos --no-eff-email --staging -d $MYDOMAIN1 -d $MYDOMAIN2`
После первого запуска проверьте наличие сертификата в каталоге certbot-etc. Если сертификат успешно получен, то следует изменить строку команду для Certbot в файле docker-compose.yml:
Удалить `--staging`, добавить `--force-renewal`

Перезапустить только Certbot можно командой `docker-compose up --force-recreate --no-deps certbot`

## Обновление сертификатов

Сертификаты Let’s Encrypt действительны в течение 90 дней, поэтому вам нужно будет настроить автоматический процесс обновления, чтобы гарантировать, что сертификаты не окажутся просроченными. Один из способов — создание задания с помощью утилиты планирования `cron`. В нашем случае мы настроим задание для `cron` с помощью скрипта, который будет обновлять наши сертификаты и перезагружать конфигурацию Nginx.

Необходимо создать задание для `chron` на выполнение скрипта `ssl_renew.sh`

Данный скрипт привязывает двоичный код docker-compose для переменной `COMPOSE` и задает параметр `--no-ansi`, который запускает команды `docker-compose` без управляющих символов ANSI. Затем он делает то же самое с двоичным файлом docker. В заключение он меняет директорию проекта на `~/wordpress` и запускает следующие команды `docker-compose`:

`docker-compose run`: данный параметр запускает контейнер certbot и переопределяет параметр command, указанный в определении службы certbot. Вместо использования субкоманды certonly мы используем здесь субкоманду renew, которая будет обновлять сертификаты, срок действия которых близок к окончанию. Мы включили параметр `--dry-run`, чтобы протестировать наш скрипт.
`docker-compose kill`: данный параметр отправляет сигнал `SIGHUP` контейнеру webserver для перезагрузки конфигурации Nginx. Дополнительную информацию об использовании этого процесса для перезагрузки конфигурации Nginx см. в этой публикации блога Docker, посвященной развертыванию официального образа Nginx с помощью Docker.
После этого он выполняет команду docker system prune для удаления всех неиспользуемых контейнеров и образов.

Сделайте ескрипт исполняемым:
`chmod +x ssl_renew.sh`
 
Далее откройте root-файл crontab для запуска скрипта обновления с заданным интервалом:
`sudo crontab -e`

Добавьте строку:
`0 12 * * * /home/sammy/wordpress/ssl_renew.sh >> /var/log/cron.log 2>&1`

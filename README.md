# sonarqube-1c

Репозиторий содержит Dockerfile и docker-compose.yaml для сборки образа и запуска Sonarqube с плагинами для 1С.
Используются СУБД PostgreSQL и оп. система Linux.

Репозиторий является идейным продолжением [репозитория](https://github.com/Daabramov/Sonarqube-for-1c-docker), но существенными различиями, которые потребовали вынесения их в отдельный репозиторий: 
Различия:
- убран лишний том postgresql, убран том data_extensions (расширения ставятся при сборке), тома сделаны именованными и создаются в одной директории с docker-compose.yaml для контроля
- настройки операционной системы приведены в соответствие с требованием оф. документации SonarQube
- нумерация образов контейнеров приведена в соответствие с нумерацией ванильных контейнеров SonarQube
- сделано развернутое руководство по осознанному выбору версий, параметрам настройки серверов с учетом RAM, а также алгоритма обновления со ссылками на официальную документацию SonarQube 

## Базовые репозиторие, используемые в коде
1. [Образ sonarqube Community Build](https://hub.docker.com/_/sonarqube)
2. [Сборка образа с включенным community-branch-plugin](https://github.com/mc1arke/sonarqube-community-branch-plugin)

## Полезные ссылки
[Официальная документация SonarQube ](https://docs.sonarsource.com/sonarqube-server) лежит в основе любых настроек рекомендуемых в различных репозиториях. Желательно проверять рекомендуемые в репозиторях настройки на соответствие документации.

## Выбор версий плагинов
Версии плагинов должны быть совместимы с выбранной версией SonarQube. Допустим, решено устанавливать версию [SonarQube 25.12.0 (25 год, 12 месяц, 0 патч)](https://docs.sonarsource.com/sonarqube-server/server-update-and-maintenance/update/release-cycle-model).
Тогда выбирается соотвествующая [версия](https://hub.docker.com/r/mc1arke/sonarqube-with-community-branch-plugin) образа SonarQube с предустановленным плагином community-branch-plugin. 
Версии плагинов sonar-bsl-plugin-community и sonar-l10n-ru выбираются исходя из таблиц совместимости с версиями SonarQube, см. описание в репозиториях плагинов [здесь](https://github.com/1c-syntax/sonar-bsl-plugin-community) и [здесь](https://github.com/1c-syntax/sonar-l10n-ru). 
Вeрсии плагинов и SonarQube задаются в файле Dockerfile:
- в наименовании образа с встроенным плагином sonarqube-community-branch-plugin, например, mc1arke/sonarqube-with-community-branch-plugin:25.12.0.117093-community
- в виде переменных окружения для плагина sonar-bsl-plugin-community: `ARG BSL_PLUGIN_VERSION=1.18.1`
- в виде переменных окружения для плагина sonar-l10n-ru: `ARG RUSSIAN_PACK=25.7`

## Сборка образа

### Локальная cборка
При необходимости, можно собрать локальный образ вручную с нужным составом плагинов. И использовать его в дальнейшем для запуска SonarQube через docker compose.
```bash
git clone https://github.com/AndrShat/Sonarqube-1c.git
cd Sonarqube-1c
docker build -t ImageName .
```

## Развертывание нового экземпляра SonarQube

```bash
# Клонируем репозиторий
git clone https://github.com/AndrShat/Sonarqube-1c.git
cd Sonarqube-1c
mkdir -p ./sonarqube_data ./sonarqube_conf
sudo chown -R 1000:1000 ./sonarqube_data ./sonarqube_conf
# Изменяем настройки системы для Elasticsearch и SonarQube 
sudo sysctl -w vm.max_map_count=524288 
sudo sysctl -w fs.file-max=131072
# не забываем добавить их /etc/sysctl.conf, чтобы они не пропали при перезагрузке
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072" | sudo tee -a /etc/sysctl.conf
# Запускаем контейнеры
docker compose up -d
```
Далее переходим по ссылке http://АйпиСервераГдеЗапущеныКонтейнеры:32772, [вводим имя пользователя admin и пароль admin](https://docs.sonarsource.com/sonarqube-server/server-installation/from-docker-image/installation-overview).

### Важные настройки docker compose
До запуска нового экземпляра SonarQube рекомендуется проверить и задать следующие настройки:
1. Сложный пароль пользователя postgresql, если есть требования безопасности:
```yaml
environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
```
2. Наименование образа контейнера из данного репозитория или локального образа, скомпилированого ранее, например:
```
image: andrshat/sonarqube-1c:26.5.0.122743-community
```
3. Порт, на котором будет слушать SonarQube:
```yaml
    ports:
      - "32772:9000"

```
4. Настройки java, соответствующие количеству RAM сервера, на котором будет работать контейнер:
Настройка для сервера с 8RAM:
```yaml
      - SONAR_WEB_JAVAOPTS=-server -Xms512m -Xmx512m -XX:+HeapDumpOnOutOfMemoryError
      - SONAR_CE_JAVAOPTS=-Xms1g -Xmx2g -XX:+HeapDumpOnOutOfMemoryError
      - SONAR_SEARCH_JAVAOPTS=-Xms1g -Xmx1g -XX:+HeapDumpOnOutOfMemoryError
```
Xms - количество памяти, выделяемое на начальном этапе для JVM, Xmx - максимальная доступная память. 
Переменные окружения настраивают одноименные компоненты сервиса SonarQube:
- Web
- Compute Engine (CE)
- Elasticsearch (ES)
Общая сумма Xmx должна быть меньше, чем объем RAM сервера, так, чтобы осталась память для операционной системы и PostgreSQL.
В данном случае для операционной системы + PostgreSQL остается 4.5GB из 8GB RAM
Описание параметров SonarQube, задаваемых через переменные среды можно посмотреть [здесь](https://docs.sonarsource.com/sonarqube-server/server-installation/system-properties/common-properties). Системные [требования](https://docs.sonarsource.com/sonarqube-server/server-installation/server-host-requirements). [Пример, настройки параметров памяти JVM](https://docs.sonarsource.com/sonarqube-server/quickstart-guide/installing-sonarqube-server-with-sql-server).

5. На уровне хоста Linux необходимо задать следующие [настройки](https://docs.sonarsource.com/sonarqube-server/server-installation/pre-installation/linux) необходимые для работы Elasticsearch:
В `/etc/sysctl.conf`
```yaml
 vm.max_map_count=524288
 fs.file-max=131072
```
Проверить текущие настройки и изменить их временно динамически можно командами:
```bash
# Проверить
sysctl vm.max_map_count
sysctl fs.file-max
# Изменить
sysctl -w vm.max_map_count=524288
sysctl -w fs.file-max=131072
```
Эти настройки будут унаследованы docker-контейнером. Настройки ulimit для максимального количества файловых дескрипторов и процессов дополнительно прописываются в docker-compose.yaml:
```yaml
    ulimits:
      nofile:
        soft: 131072
        hard: 131072
      nproc:
        soft: 8192
        hard: 8192

```
6.Настройки томов задаются в docker-compose.yaml стандартным образом. Том, который отвечает за сохранение плагинов закомментирован, так как плагины добавляются на этапе создания образа. Если планируется добавлять плагины вручную, можно раскомментировать том sonarqube_extentions. В этом случае все плагины необходимо настроить и добавить вручную.


## Обновление
При обновлении необходимо прочитать [раздел про обновление](https://docs.sonarsource.com/sonarqube-server/server-update-and-maintenance/update) документации SonarQube.
Основные моменты:
1. Перед обновлением необходимо проверить, можно ли обновляться на выбранную версию сразу или необходимо обновление на промежуточные версии. Это можно сделать [здесь.](https://docs.sonarsource.com/sonarqube-server/10.7/server-upgrade-and-maintenance/upgrade/upgrade-the-server/determine-path). Для каждой версии (включая промежуточные), на которую производится обновление выполняются все шаги обновления.
**Важно:** В калькуляторе пути обновления допущена ошибка. Указано, что с версии 25.12 можно обновлятся на любую версию 26.X. Однако, [необходимо сперва обновиться на версию 26.1](https://community.sonarsource.com/t/update-sonarqube-community-edition-25-12-to-26-3/181176).
2. Перед обновлением необходимо сделать [обслуживание базы данных PostgreSQL](https://docs.sonarsource.com/sonarqube-server/server-update-and-maintenance/update/pre-update-steps):
```sql
VACUUM FULL;
REINDEX DATABASE sonar;
ANALYZE;
```
3. Cделать бэкап томов контейнера и базы данных (лучше на остановленном SonarQube);
4. Собрать образ с новой версией SonarQube и совместимыми плагинами;
5. Остановить SonarQube (`docker compose down`). Прописать ссылку на новый образ в docker-compose.yaml, запустить SonarQube (docker compose up), следить за возможными ошибками в логах.
6. Если ошибок в логах не было, то перейти на страницу http://SonarDomain:32772/setup и выполнить обновление (миграцию) базы данных.
7. Зайти в SonarQube под администратором и дождаться выполнения все регламентных заданий по обновлению.


## Troubleshooting
См. [ссылку](https://docs.sonarsource.com/sonarqube-server/quickstart-guide/installing-sonarqube-server-with-sql-server)



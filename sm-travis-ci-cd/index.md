# Внедрение CI\CD для проекта Screen-Monitoring с помощью http://travis-ci.org

## Содержание

1. Введение
2. Про Travis
3. Подготовка
4. Настраиваем CI
5. Настраиваем CD
6. Что получилось
7. Список литературы

## Введение
Это история о том, как мы настраивали Continuous delivery и Сontinuous integration для нашего Open Source проекта [Screen-Monitoring](https://github.com/maddevsio/screen-monitoring "Github")

Основная концепция этого проекта - мониторинг для IT команды.

Screen-Monitoring должен уметь собирать "любую" информацию и показывать ее различными способами. Проект написан на языке GO и является абсолютно бесплатным.

Сейчас в архитектуре проекта есть две сущности:
* Dashboard - центральная точка взаимодействия между пользователями и агентами
* Agents - собственно агенты, которые собирают, агрегируют и показывают информацию юзеру через Dashboard

Я хочу осветить в этой статье как мы реализовали тесты, сборку и поставку на боевой сервер центральной части проекта Screen-Monitoring -  [Dasboard](https://github.com/maddevsio/screen-monitoring "Github").

Для начала определимся, что же такое Continuous delivery и Сontinuous integration?

Continuous delivery или непрерывная поставка ПО — рекомендации, нацеленные на поставку ПО как можно быстрее и чаще в боевое окружение (на работающий сервер) стандартным и полностью автоматизированным способом.

Сontinuous integration или непрерывная интеграция — это практика разработки программного обеспечения, которая заключается в слиянии рабочих копий в общую основную ветвь разработки несколько раз в день и выполнении частых автоматизированных сборок проекта для скорейшего выявления и решения интеграционных проблем.

Так как наш проект является Open Source и находится в открытом доступе, мы решили остановится на бесплатном решении CI\CD - https://travis-ci.org/

## Про Travis

Travis-ci очень любопытный проект для организации CI\CD, тесно интегрированный с github.com. Он позволяет организовать тестирование и сборки програмного обеспечения использующего Github в качестве хостинга исходного кода. Сервис поддерживает работу с большим количеством языков - C, C++, D, JavaScript, Java, PHP, Python и Ruby. Поддерживает большое колличество сторонних программ и скриптов(git, docker, bash), а также множество возможностей для деплоя сборок на различные Cloud Services (AWS, Google Cloud Storage, Heroku, S3).

Еще немаловажная фича Travis-ci это поддержки шифрованных переменных и шифрованных файлов лежащих в репозитории проекта, это может потребоватся для сборки или поставки приложения на сервер не беспокоясь о том, что пароли от БД или другие секретные данные могут попасть не в те руки.

## Подготовка

Для начала определимся с реализацией.

На входе у нас есть:
* проект написанный на GO
* репозиторий на Github
* написанные тесты

Что должно получится:
* готовые Docker Images на https://hub.docker.com/
* деплой успешной сборки на боевой сервер

Для начала нужно получить в репо проекта роль "Admin", это необходимо для дальнейшей интеграции с Travis-ci.

Далее желательно завести в проекте пустой файл ".travis.yml" и задеплоить его на Github.

Затем необходимо авторизоватся в Travis-CI через OAuth(github) и разрешить ему просматривать текущий Github аккаунт. В настройках github (https://github.com/settings/applications - Authorized applications) появится разрешенное приложение Travis-CI.

Дальше на главной странице https://travis-ci.org/ появляется список репозиториев разрешенных организаций и твои собственные репозитории (но только в том случае если они не приватные, для приватных придется воспользоватся платной подпиской на https://travis-ci.com/).

Являясь участником организации в Github, ты можешь видеть активность добавленных в Travis-CI репозиториев этой организации, но не можешь на них повлиять если твои права ниже (Owner или Admin). Например добавить\удалить переменные, запустить билд.

В Github правами репозиториев можно рулить на уровне Organizations или Teams (если над проектом работает множество специалистов).

Как только участник группы получает уровень Owner\Admin, ему следует зайти в Travis-CI и нажать в настройках Sync Account чтобы применились настройки доступа.

Теперь можно интегрировать репозиторий проекта в Travis-CI. Чтобы забрать возможность правки проекта в Travis-CI нужно понизить роль у участника, либо удалить его из Организации.

Выбираем проект который хотим запустить в Travis-CI и включаем в "General Settings" следующее:
* Build only if .travis.yml is present
* Build pushes
* Build pull requests

Теперь, можно запустить первую сборку, но она пока еще проходить не будет, так как travis.yml у нас пока пуст.

Последний пункт в подготовке это создание аккаунта на hub.docker.com. С этим не должно возникнуть больших проблем. Я дополнительно создал компанию maddevsio и пригласил в нее своих колег.

## Настраиваем CI

На этапе "CI" после любого push в репо мы должны получить какой либо артефакт, это может быть отчет по пройденным тестам, билд программы или как в нашем случае Docker Image лежащий на https://hub.docker.com/r/maddevsio/sm-dashboard/.

Сборку Docker Image, нужно начинать после успешного прохожденя Unit Тестов и успешной сборки приложения.

Вот с этого и начнем, добавим в ".travis.yml" начальный конфиг:

```
# Здесь мы указываем нужный язык и
# по необходимости конкретные его версии
language: go
go:
- 1.7.3
- tip
# Настраиваем окружение
before_install:
  - ". $HOME/.nvm/nvm.sh"
  - nvm install stable
  - nvm use stable
  - npm install
# Запускаем шаг тестов и сборки, если на этом
# шаге что-то пойдет не так, Travis пометит сборку
# как Failed
script:
  - go test -v ./dashboard/
  - go build -v
  - npm run build-production
```

Вот и все, теперь после каждого push в репо у нас будут запускатся тесты в Travis-CI.

Однако кроме тестов и сборки проекта хотелось бы получить готовый Image для Docker.

Для этого мы создали файл сборки Image "Dockerfile_Travis" в репо:
```
FROM golang:1.7

MAINTAINER Andreev Vlad <andreevlad@gmail.com>

COPY ./screen-monitoring /screen-monitoring/
COPY ./dashboard /screen-monitoring/dashboard
COPY ./public /screen-monitoring/public

WORKDIR /screen-monitoring

EXPOSE 8080

CMD ["./screen-monitoring"]

```
И написали bash скрипт сборки, который положили в отдельный репо, так как расчитываем использовать его для сборки других проектов:
https://raw.githubusercontent.com/maddevsio/travis-push-to-docker/master/sm-push.sh

```
#!/bin/bash

function tag_and_push {
	if [ -n "$1" ] && [ -n "$IMAGE_NAME" ]; then
		echo "Pushing docker image to hub tagged as $IMAGE_NAME:$1"
		docker build -t $IMAGE_NAME:$1 -t $IMAGE_NAME -f Dockerfile_Travis .
		docker push $IMAGE_NAME
		docker push $IMAGE_NAME:$1
	fi
}

VERSION_TAG=v.$TRAVIS_BUILD_NUMBER

if [ "${TRAVIS_GO_VERSION}" = "${GO_FOR_RELEASE}" ]; then

cat > ~/.dockercfg << EOF
{
  "https://index.docker.io/v1/": {
    "auth": "${HUB_AUTH}",
    "email": "${HUB_EMAIL}"
  }
}
EOF

	tag_and_push $VERSION_TAG
else
	echo "No image to build"
fi
```
Скрипт проверяет версию GO в текущей сборке, это нужно если есть необходимось тестирования на разных версиях языка, например - 1.7.3 и самой последней:
```
go:
- 1.7.3
- tip
```
далее скрипт берет номер текущей сборки (VERSION_TAG=v.$TRAVIS_BUILD_NUMBER) и подставляет его в команду "docker build" в виде TAG метки названия Docker Image, получается примерно так:
```
docker build maddevsio/sm-dashboard:v.32
docker push maddevsio/sm-dashboard:v.32
```

Чтобы запушить Image на hub.docker.com нужно воспользоватся шифрованными переменными в Travis-CI и указать следующие переменные:
```
HUB_AUTH
HUB_EMAIL
```

Это данные аутентификации на hub.docker.com. Получить HUB_AUTH можно вызвав утилиту "docker login" и ввести свои данные от учетки с hub.docker.com, после этого появится файлик $HOME/.docker/config.json в котором лежат нужные нам данные


Итоговый конфиг для Travis-CI получился таким:
```
language: go
go:
- 1.7.3
- tip
env:
  global:
  - GO_FOR_RELEASE=1.7.3
  - IMAGE_NAME=maddevsio/sm-dashboard
services:
  - docker
before_install:
  - ". $HOME/.nvm/nvm.sh"
  - nvm install stable
  - nvm use stable
  - npm install
script:
  - go test -v ./dashboard/
  - go build -v
  - npm run build-production
  - curl https://raw.githubusercontent.com/maddevsio/travis-push-to-docker/master/sm-push.sh | bash
```

Теперь после успешного прохождения тестов и сборки билда, запускается скрипт сборки Docker image куда копируется собранное ранее приложение и заливается в hub.docker.com

## Настраиваем CD

На этом этапе нужно научить Travis-CI деплоить получившийся Docker Image на сервер.

В нашем случае сервер это обычная виртуалка с Debian Linux на борту и заранее установленным Docker.

После прохождения тестов и сборки проекта мы получили готовый [Docker Image](https://hub.docker.com/r/maddevsio/sm-dashboard/ "maddevsio/sm-dashboard"), с ним и продолжим дальнейшие работы.

Деплоить с помощью Docker можно несколькими путями:
1. Через ssh под каким либо пользователем передавать команды демону Dockerd. Схема такая - Travis соединяется с хостом через ssh и выполняет команды (docker run\rm ...). Минус этого способа - необходимость передачи Тревису id_rsa ключ, в этом случае на помощь придет возможность шифровать файлы. Однако шифрованный ключ должен хранится в репозитории проекта с которым работает Travis, что для крупных и закрытых проектов может быть не приемлемо.
2. Прокинуть TCP socker демона Dockerd наружу, например через nginx, DNAT или указать демону Dockerd внешний IP. В этом случае встает вопрос защиты Dockerd. Можно зарезать по IP, но в нашем случае мы не знаем с каких IP может прийти соединение. Еще можно попробовать воспользоватся TLS, на хосте с Dockerd создаются crt ключи, указывается директория хранения ключей и необходимость их использования для аутентификации. Дальше этими же сертификатами подписывается сертификат клиента, которые передаются Travis-CI. Тут все усложняется необходимостью работы с сертификатами, но в целом думаю это наилучший и наиболее правильный сценарий для работы с Dockerd удаленным клиентам.
3. У Dockerd есть HTTP_API. Можно включить HTTP_API, кинуть порт на loopback интерфейс и получить к нему доступ через nginx с HTTP_AUTH. Есть два минуса, HTTP_AUTH не достаточно надежен, а запросы которые придется передавать с консоли получаются очень громоздкими, хотя в целом если один раз написать билд план и скрипты к нему, этот вариант тоже вполне можно использовать.


Я решил использовать первый вариант как самый быстрый в реализации.

На сервере нужно создать пользователя (в нашем случае имя пользователя будет - sm-docker) для деплоев:


```
adduser --home /srv/docker/sm-docker sm-docker
usermod -a -G docker sm-docker

ssh-keygen
cp id_rsa.pub authorized_keys
chmod 600 authorized_keys
change authorized_keys << command="docker $SSH_ORIGINAL_COMMAND",no-port-forwarding,no-agent-forwarding,no-pty ssh-rsa AAAAB3NzaC1yc2EA..... .ssh/id_rsa

```

С помощью директивы `command` в файле `authorized_keys` я ограничил этому пользователю запуск каких либо команд через ssh кроме утилиты docker, которая принимает в качестве аргументов любые данные введенные с удаленного хоста, например команда:
```
ssh -o StrictHostKeyChecking=no -i ./sm-docker-key sm-docker@sm.maddevs.io "rm -f screen-monitoring"

```
запустит на удаленном хосте:

```
docker rm -f screen-monitoring
```


Созданный id_rsa ключ для пользователя sm-docker я добавил в репо проекта предварительно зашифровав его утилитой travis, при шифровании утилита сама поправила файл .travis.yml добавив туда функцию расшифровки файла, а также добавила переменные для расшифровки в build plan нашего проекта на travis-ci.org

```
travis encrypt-file  ./sm-docker-key --add
```

После этой процедуры оригинальный файл ключа надо удалить.

Для самого деплоя воспользуемся шагом Deploy в Travis-CI:
```
language: go
go:
- 1.7.3
- tip
env:
 global:
 - GO_FOR_RELEASE=1.7.3
 - IMAGE_NAME=maddevsio/sm-dashboard
services:
 - docker
before_install:
 - openssl aes-256-cbc -K $encrypted_abfc74a8332b_key -iv $encrypted_abfc74a8332b_iv -in sm-docker-key.enc -out ./sm-docker-key -d
 - ". $HOME/.nvm/nvm.sh"
 - nvm install stable
 - nvm use stable
 - npm install
script:
 - go test -v ./dashboard/
 - go build -v
 - npm run build-production
 - curl https://raw.githubusercontent.com/maddevsio/travis-push-to-docker/master/sm-push.sh | bash
deploy:
 provider: script
 skip_cleanup: true
 script: chmod 600 sm-docker-key &&
 ssh -o StrictHostKeyChecking=no -i ./sm-docker-key sm-docker@sm.maddevs.io "pull $IMAGE_NAME:v.$TRAVIS_BUILD_NUMBER" &&
 ssh -o StrictHostKeyChecking=no -i ./sm-docker-key sm-docker@sm.maddevs.io "rm -f screen-monitoring" || true &&
 ssh -o StrictHostKeyChecking=no -i ./sm-docker-key sm-docker@sm.maddevs.io "run -d --restart=always --name=screen-monitoring -p 127.0.0.1:9080:8080 -v /srv/docker/sm-docker/screen-monitoring/screen_monitoring.db:/screen-monitoring/screen_monitoring.db $IMAGE_NAME:v.$TRAVIS_BUILD_NUMBER"
 on:
 go: '1.7.3'

```

Теперь после каждого push в репо на github запускается build plan в - Travis-ci.
При успешном прохождении тестов и сборки docker image запускается шаг Deploy.
В шаге Deploy на хост sm.maddevs.io пулится image с номером текушей сборки, удаляется старый контейнер screen-monitoring и запускается новый.

Благодаря Travis-CI получилось организовать легкую и бесплатную реализацию CI\CD.


## Полезные ссылки

Если интересно поглубже изучить применяемые в статье технологии, можно почитать следующее:
* https://github.com/maddevsio/screen-monitoring - Наш проект распределенного мониторинга, агрегации и представления данных для IT комманд.
* https://docs.travis-ci.com/ - Документация Travis-CI
* https://docs.docker.com/ - Документация Docker
* https://www.linux.com/learn/integrating-docker-hub-your-application-build-process - Integrating Docker Hub In Your Application Build Process
* https://www.linux.com/learn/how-automate-web-application-testing-docker-and-travis - How to Automate Web Application Testing With Docker and Travis
* https://www.linux.com/learn/automatically-deploy-build-images-travis - Automatically Deploy Build Images with Travis
* https://sheerun.net/2014/05/17/remote-access-to-docker-with-tls/ - Remote access to Docker with TLS
* http://serverfault.com/questions/749474/ssh-authorized-keys-command-option-multiple-commands - SSH authorized_keys command option: multiple commands?
* https://cybermashup.com/2013/05/14/restrict-ssh-logins-to-a-single-command/ - Restrict SSH logins to a single command
* https://habrahabr.ru/post/140344/ - Что такое travis-ci.org и с чем его едят?
* https://ru.wikipedia.org/wiki/%D0%9D%D0%B5%D0%BF%D1%80%D0%B5%D1%80%D1%8B%D0%B2%D0%BD%D0%B0%D1%8F_%D0%B8%D0%BD%D1%82%D0%B5%D0%B3%D1%80%D0%B0%D1%86%D0%B8%D1%8F - Непрерывная интеграция


Автор статьи: [Андреев Владислав](https://github.com/andreevlad)

Статья написана в рамках работы по интеграции Travis-CI в компании [Maddevs](http://maddevs.io)

Хочу выразить отдельную благодарность за помощь с интеграцией [Кареву Генадию](https://github.com/pendolf)

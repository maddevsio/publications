## Запускаем процесс CI для андроида с помощью Docker, Jenkins и Pipeline за $0

Для автоматизации различных воркфлоу в нашей команде давно и активно используется [Jenkins](https://jenkins.io/) в связке с его очень мощным плагином [Pipeline](https://wiki.jenkins-ci.org/display/JENKINS/Pipeline+Plugin). Пайплайн позволил нам почти полностью уйти от настройки Дженкинса через вебморду и рулить CI/CD воркфлоу проектов groovy-скриптами + scm. [Docker](https://www.docker.com/) в зависимости от проекта служит нам соборочным и тестовым окружениями, репозиторием артефактов, средством поставки на стейджинг и прод итд. Ниже будет рассмотрен пример пайплайна для непрерывной интеграции андроид проекта.

![Pipeline](https://github.com/maddevsio/publications/blob/master/pipeline-android/img/pipeline.png)

### Подготовка
Все сервисы, включая дженкинс будут запущены в контейнерах. На хост необходимо [установить докер](https://docs.docker.com/engine/installation/) и запустить [Дженкинс](https://hub.docker.com/_/jenkins/)
```bash
docker run -d -p 8080:8080 -p 50000:50000 -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock jenkins
```
Помимо стандартных предустановленных плагинов, необходимы:
* Pipeline
* CloudBees Docker
* EnvInject
* GitHub Authentication
* HTML Publisher
* JenkinsLint
* Slack Notification
* JUnit
* ChuckNorris

UI-тесты будут проходить на реальном телефоне, который подключен через usb к хосту. Для постоянного adb соединения с ним запустим еще один [контейнер](https://hub.docker.com/r/sorccu/adb/)
```bash
docker run -d --privileged -v /dev/bus/usb:/dev/bus/usb --name adbd sorccu/adb
```

### Окружение
В корень проекта нужно будет добавить Dockerfile с тестовым окружением. За основу были взят этот [Dockerfile](https://hub.docker.com/r/jacekmarchwicki/android/) и слегка доработан:
```Dockerfile
FROM ubuntu:14.04

# Install java8
RUN apt-get update && apt-get install -y software-properties-common && apt-get clean && rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/*
RUN add-apt-repository -y ppa:webupd8team/java
RUN echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | /usr/bin/debconf-set-selections
RUN apt-get update && apt-get install -y oracle-java8-installer && apt-get clean && rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Install Deps
RUN dpkg --add-architecture i386 && apt-get update \
    && apt-get install -y --force-yes expect git wget locales \
       libc6-i386 lib32stdc++6 lib32gcc1 lib32ncurses5 lib32z1 python curl \
    && apt-get clean && rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Configure locale to utf8
RUN localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8

ARG JENKINS_HOME=/var/jenkins_home
ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000

# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN groupadd -g ${gid} ${group} \
    && useradd -d "$JENKINS_HOME" -u ${uid} -g ${gid} -m -s /bin/bash ${user}

# Install Android SDK
ARG ANDROID_SDK_VERSION=r24.4.1
ARG ANDROID_BUILD_TOOLS_VERSION=23.0.3
ARG ANDROID_API_LEVELS=android-23

ENV ANDROID_HOME /opt/android-sdk-linux
ENV PATH ${PATH}:${ANDROID_HOME}/tools:${ANDROID_HOME}/platform-tools

RUN cd /opt && \
    wget --output-document=android-sdk.tgz --quiet http://dl.google.com/android/android-sdk_${ANDROID_SDK_VERSION}-linux.tgz && \
    tar xzf android-sdk.tgz && \
    rm -f android-sdk.tgz && \
    chown -R root:root android-sdk-linux

RUN echo y | android update sdk --no-ui -a --filter tools,platform-tools,${ANDROID_API_LEVELS},build-tools-${ANDROID_BUILD_TOOLS_VERSION},extra-android-support,extra-android-m2repository,extra-google-m2repository

USER jenkins
```
Тут помимо установки oracle-java8 мы настраиваем локаль, заводим пользователя для дженкинс и задаем необходимые версии для API и SDK. Дефолтные значения можно перезадать при сборке контейера через build-arg.

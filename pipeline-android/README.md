## Запускаем процесс CI для андроида с помощью Docker, Jenkins и Pipeline за $0

Для автоматизации различных воркфлоу в нашей команде давно и активно используется [Jenkins](https://jenkins.io/) в связке с его очень мощным плагином [Pipeline](https://wiki.jenkins-ci.org/display/JENKINS/Pipeline+Plugin). Пайплайн позволил нам почти полностью уйти от настройки Дженкинса через вебморду и рулить CI/CD воркфлоу проектов groovy-скриптами + scm. [Docker](https://www.docker.com/) в зависимости от проекта служит нам соборочным и тестовым окружениями, репозиторием артефактов, средством поставки итд. Ниже будет рассмотрен пример пайплайна для непрерывной интеграции андроид проекта.

![Pipeline](https://github.com/maddevsio/publications/blob/master/pipeline-android/img/pipeline.png)

### Подготовка
Все сервисы, включая дженкинс будут запущены в контейнерах. На хост необходимо [установить докер](https://docs.docker.com/engine/installation/) и запустить [Дженкинс](https://hub.docker.com/_/jenkins/)
```bash
docker run -d -p 8080:8080 -p 50000:50000 -v /usr/bin/docker:/usr/bin/docker -v /var/run/docker.sock:/var/run/docker.sock jenkins
```
UI-тесты будут проходить на реальном телефоне, который подключен через usb к хосту. Для постоянного adb соединения с ним запустим еще один [контейнер](https://hub.docker.com/r/sorccu/adb/)
```bash
docker run -d --privileged -v /dev/bus/usb:/dev/bus/usb --name adbd sorccu/adb
```

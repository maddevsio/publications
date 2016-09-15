## Запускаем процесс CI для андроида с помощью Docker, Jenkins и Pipeline за $0

Для автоматизации различных воркфлоу в нашей команде давно и активно используется [Jenkins](https://jenkins.io/) в связке с его очень мощным плагином [Pipeline](https://wiki.jenkins-ci.org/display/JENKINS/Pipeline+Plugin). Пайплайн позволил нам почти полностью уйти от настройки Дженкинса через вебморду и рулить CI/CD воркфлоу проектов groovy-скриптами + scm. [Docker](https://www.docker.com/) в зависимости от проекта служит нам соборочным и тестовым окружениями, репозиторием артефактов, средством поставки итд. Ниже будет рассмотрен пример пайплайна для непрерывной интеграции андроид проекта

![Pipeline](https://github.com/maddevsio/publications/blob/master/pipeline-android/img/pipeline.png)

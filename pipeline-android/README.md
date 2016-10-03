## Запускаем процесс CI для андроида с помощью Docker, Jenkins и Pipeline за $0

Для автоматизации различных воркфлоу в нашей команде давно и активно используется [Jenkins](https://jenkins.io/) в связке с его очень мощным плагином [Pipeline](https://wiki.jenkins-ci.org/display/JENKINS/Pipeline+Plugin). Пайплайн позволил нам почти полностью уйти от настройки Дженкинса через вебморду и рулить CI/CD воркфлоу проектов groovy-скриптами + scm. [Docker](https://www.docker.com/) в зависимости от проекта служит нам сборочным и тестовым окружениями, репозиторием артефактов, средством поставки на стейджинг и прод и т.д. Ниже будет рассмотрен пример пайплайна для непрерывной интеграции андроид проекта.

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
Для проверки работы с телефоном введем:
```bash
# to print connected devices
docker run --rm -ti --net container:adbd sorccu/adb adb devices
List of devices attached
1f75b250        device
# to turn screen on
docker run --rm -ti --net container:adbd sorccu/adb adb shell input keyevent 26
# to swipe to unlock
docker run --rm -ti --net container:adbd sorccu/adb adb shell input swipe 400 800 400 400
```

### Окружение
В репо проекта нужно будет добавить Dockerfile с тестовым окружением. Мы взяли [Dockerfile отсюда](https://hub.docker.com/r/jacekmarchwicki/android/) и слегка доработали:
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

# Jenkins is run with user `jenkins`, uid = 1000
ARG JENKINS_HOME=/var/jenkins_home
ARG user=jenkins
ARG group=jenkins
ARG uid=1000
ARG gid=1000

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
Тут помимо установки oracle-java8 мы настраиваем локаль, заводим пользователя для дженкинс и задаем необходимые версии для API и SDK. Дефолтные значения можно перезадать при сборке контейнера через --build-arg.

### Пайплайн
За основу пайплайн-скрипта был взят данный [Jenkinsfile](http://flyingtophat.co.uk/blog/2016/07/07/continuous-integration-for-android-with-jenkins-docker-and-aws.html), однако у нас инструментальные тесты проходят локально. Также был добавлен дополнительный стейдж, который ставит тег в репо при успешном выполнении тестов, + отправляется уведомление в слак:
```Groovy
#!groovy​
try {
  node() {
    notifyBuild('STARTED')
    deleteDir()

    stage 'Checkout'
    checkout scm

    stage 'Create Env'
    def buildEnv = docker.build('android-sdk', ".")
    buildEnv.inside('--net container:adbd') {
      sh '''mkdir -p ?/.android
            keytool -genkey -v -keystore ?/.android/debug.keystore -storepass android -alias androiddebugkey -keypass android -dname "CN=Android Debug,O=Android,C=US"
         '''

      stage 'Build'
      sh './gradlew clean assembleDebug'
      archive 'app/build/outputs/**/app-debug.apk'

      stage 'Quality'
      sh './gradlew lint'
      stash includes: '*/build/outputs/lint-results*.xml', name: 'lint-reports'

      stage 'Test (unit)'
      try {
        sh './gradlew test'
      } catch (err) {
        currentBuild.result = 'UNSTABLE'
      }
      stash includes: '**/test-results/**/*.xml', name: 'junit-reports'

      stage 'Test (device)'
      try {
        sh './gradlew connectedDebugAndroidTest'
      } catch (err) {
        currentBuild.result = 'UNSTABLE'
      }
      stash includes: 'app/build/reports/androidTests/connected/*.html', name: 'ui-reports'
    }

    stage 'Tag/Push'
    if (currentBuild.result == 'UNSTABLE') {
      echo 'Build is unstable. Skip tagging repo'
    } else {
      tagGitRepo()
    }
  }

  stage 'Report'
  node() {
    deleteDir()

    unstash 'junit-reports'
    step([$class: 'JUnitResultArchiver', testResults: '**/test-results/**/*.xml'])

    unstash 'lint-reports'
    step([$class: 'LintPublisher', canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', healthy: '', pattern: '*/build/outputs/lint-results*.xml', unHealthy: ''])

    unstash 'ui-reports'
    publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: true, reportDir: 'app/build/reports/androidTests/connected', reportFiles: 'index.html', reportName: 'UI Tests Report'])
  }
} catch (e) {
  currentBuild.result = "FAILED"
  throw e
} finally {
  notifyBuild(currentBuild.result)
}

def tagGitRepo () {
  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'MyID', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
      sh("git tag -a some_tag -m 'Jenkins'")
      sh('git push https://${GIT_USERNAME}:${GIT_PASSWORD}@<REPO> --tags')
  }
}

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorName = 'danger'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'warning'
  } else if (buildStatus == 'SUCCESSFUL') {
    color = 'good'
  } else if (buildStatus == 'UNSTABLE') {
    color = 'warning'
  } else {
    color = 'danger'
  }

  slackSend (color: color, message: summary)
}
```

Jenkinsfile так же кладется в репозиторий проекта, после чего в Дженкинсе создаем новый Pipeline проект и в настройках выбираем "Pipeline script from scm"
![Pipeline script from scm](https://github.com/maddevsio/publications/blob/master/pipeline-android/img/jenkinsfile-from-scm.png)

в разделе **Build Triggers** выбираем Poll SCM и задаем периодичность. После чего при мердже или коммите в мастер наблюдаем подобную картину:
![Stage View](https://github.com/maddevsio/publications/blob/master/pipeline-android/img/stage-view.png)

### Полезные ссылки
* [Pipeliene](https://jenkins.io/solutions/pipeline/)
* [Pipeline Examples](https://github.com/jenkinsci/pipeline-examples)
* [Parallelism with Jenkins Pipeline](https://www.cloudbees.com/blog/parallelism-and-distributed-builds-jenkins)
* [Pipeline Notifications](https://jenkins.io/blog/2016/07/18/pipline-notifications/)
* [Top 10 Best Practices for Jenkins Pipeline Plugin](https://www.cloudbees.com/blog/top-10-best-practices-jenkins-pipeline-plugin)
* [Docker Pipeline Plugin](https://go.cloudbees.com/docs/cloudbees-documentation/cje-user-guide/chapter-docker-workflow.html)
* [CLOUDBEES + DOCKER](http://www.levvel.io/blog-post/building-devops-artifact-pipeline-for-docker-containers)
* [Continuous integration for Android with Jenkins, Docker and AWS](http://flyingtophat.co.uk/blog/2016/07/07/continuous-integration-for-android-with-jenkins-docker-and-aws.html)
* [Oracle Java + Docker](http://blog.takipi.com/running-java-on-docker-youre-breaking-the-law/)

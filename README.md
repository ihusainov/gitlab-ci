# gitlab-ci
gitlab-ci общая информация

Запуск в сборки docker образа в gitlab-runner
------

Для сборки образов необходимо что бы в контейнере где выполняются сборки был docker.
Для сборки в pipeline нам понадобится слудующее:

1) Прочитать статью  https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci

Существует два основных подхода: фактическая установка демона Docker внутри Docker и совместное использование демона хоста в контейнерах.

2) Использовать docker:dind как сервис. Просто добавьте docker:dind в качестве общего сервиса в файл gitlab-ci.yml
 и используйте образ docker:latest для своих задач.

```
image: docker:latest
services:
  - docker:dind
```

Профит:
- прост в настройке.
- простота запуска — ваши исходные коды по умолчанию доступны для вашего задания в cwd, поскольку они передаются непосредственно в ваш Docker Runner.

Минусы: 
- необходимо настроить реестр Docker для этой службы, иначе ваши файлы Dockerfiles будут создаваться с нуля каждый раз при запуске конвейера.
- это неприемлемо, т. к. может занять больше часа в зависимости от количества имеющихся у вас контейнеров.

3) Совместное использование /var/run/docker.sock демона хоста Docker
Используем Docker-cli с помощью демона Docker хоста с общим доступом к сокету /var/run/docker.sock,
добавив его в файл на виртуалке с gitlab-runner /etc/gitlab-runner/config.toml.
Таким образом, мы сделали демон docker нашей машины доступным для docker cli внутри контейнеров.
Обратите внимание: в этом случае вам не нужен привилегированный режим.

Профит:
- Нам не нужен специальный реестр образов docker, потому что мы используем реестр образов между всеми контейнерами.
- В этом случае нужно как-то передавать исходники в контейнеры, потому что вы монтируете их только к экзекьютеру докера
  а не к контейнерам, запускаемым из него.
  Как вариант использовать следующую комманду:
   git clone $CI_REPOSITORY_URL --branch $CI_COMMIT_REF_NAME --single-branch /project

4) Как использовать docker in docker в kubernetes
   В каталоге на хосте где у вас установлен helm создать файл values.yaml и добавить в него следующие строки:

```
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "docker:latest"
        privileged = true
        [[runners.kubernetes.volumes.host_path]]
          name = "docker"
          mount_path = "/var/run/docker.sock"
```

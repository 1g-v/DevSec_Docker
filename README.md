# Лабораторная работа. SDL_Docker
## Задание

Развернуть окружение веб-сайта с использованием базы данных, настроить постоянное хранение и создать сетевое окружение. Проверить получившийся контейнер и образ на безопасность. Разобрать вывод сканеров безопасности и предложить меры по их исправлению.

## Структура отчета
1. [Установка Docker](https://github.com/1g-v/SDL_Docker/edit/main/README.md#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-docker)
2. [Подготовка образов и запуск контейнеров]()
3. Сканирование образов на уязвимости

## Установка Docker

Следуя [официальному гайду](https://docs.docker.com/engine/install/ubuntu/) выполняем установку.

<p align="center">
<img src="https://user-images.githubusercontent.com/36410375/234104007-b22b4178-a6ab-4047-8371-35567555f151.gif">
</p>

## Подготовка образов и запуск контейнеров

### Dockerfile для Nginx

Для запуска веб-сервера Nginx в контейнере необходимо создать Dockerfile со следующим содержанием:

	FROM nginx:latest
	COPY index.html /usr/share/nginx/html/

В этом Dockerfile мы указываем базовый образ `nginx:latest` и копируем файл `index.html` в директорию `/usr/share/nginx/html/` внутри контейнера. 

### Dockerfile для Redis

Для запуска базы данных Redis в контейнере необходимо создать Dockerfile со следующим содержанием:

	FROM redis:latest
	VOLUME /data
	WORKDIR /data
	EXPOSE 6379
	CMD ["redis-server"]

В этом Dockerfile мы указываем базовый образ `redis:latest`, настраиваем Redis на сохранение данных в `/data`, открываем порт 6379 для возможности подключения к базе данных и выполняем команду запуска сервера.

### Файл docker-compose

Для запуска обоих контейнеров мы будем использовать docker-compose, который позволяет запускать несколько контейнеров одновременно. 

В файле `docker-compose.yml` мы опишем два сервиса: Nginx и Redis. 

```yaml
version: '3'

services:
  web:
    build:
      context: ./nginx
      dockerfile: Dockerfile.nginx
    ports:
      - "80:80"
    networks:
      - true-network
    depends_on:
      - redis

  redis:
    build:
      context: ./redis
      dockerfile: Dockerfile.redis
    ports:
      - "6379:6379"
    networks:
      - true-network
    volumes:
      - redis-data:/data

networks:
  true-network:

volumes:
  redis-data:

```

В этом файле мы указываем версию `3` для docker-compose. Далее определяем два сервиса: `nginx` и `redis`. 

Для каждого сервиса мы указываем `build`, где `context` указывает на директорию с Dockerfile для данного сервиса, а `dockerfile` указывает на имя Dockerfile. 

Для того, чтобы к веб-серверу можно было обратиться через браузер, пробросим порт 80 для сервиса `nginx`. Доступ к серверу базы данных обеспечивается засчет проброса порта 6379 для сервиса `redis`.

### Запуск контейнеров

Для запуска контейнеров введите команду `docker-compose up` в терминале. Docker соберет образы и запустит контейнеры. 

После запуска можно обратиться к веб-серверу Nginx по адресу `http://localhost/` и убедиться в его доступности. 
<details>
<summary>Screenshot</summary>

![image](https://user-images.githubusercontent.com/36410375/234109209-255ddd69-81cb-4937-81ee-30b6f0f21de1.png)
</details>

## Сканирование образов на уязвимости

Получим список запущенных контейнеров и с помощью утилиты **Trivy** поочередно просканируем на наличие уже известных уязвимостей каждый из Docker-образов .

	$ docker ps
	CONTAINER ID   IMAGE           COMMAND                  CREATED       STATUS       PORTS                                       NAMES
	28e350829b99   nginx:latest   "/docker-entrypoint.…"   4 hours ago   Up 4 hours   0.0.0.0:80->80/tcp, :::80->80/tcp           project_web_1
	96404310a0e3   redis:latest   "docker-entrypoint.s…"   4 hours ago   Up 4 hours   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   project_redis_1

### Сканирование nginx

Теперь с помощью команды `trivy image` и флагом `--severity HIGH,CRITICAL` получим список известных уязвимостей со значением `HIGH` и `CRITICAL` для образа `nginx:latest`.
<details>
<summary>Output</summary>

```
$ trivy image --format table --severity HIGH,CRITICAL nginx:latest
2023-04-24T14:52:03.151-0400	INFO	Vulnerability scanning is enabled
2023-04-24T14:52:03.151-0400	INFO	Secret scanning is enabled
2023-04-24T14:52:03.151-0400	INFO	If your scanning is slow, please try '--scanners vuln' to disable secret scanning
2023-04-24T14:52:03.151-0400	INFO	Please see also https://aquasecurity.github.io/trivy/dev/docs/secret/scanning/#recommendation for faster secret detection
2023-04-24T14:52:05.825-0400	INFO	Detected OS: debian
2023-04-24T14:52:05.826-0400	INFO	Detecting Debian vulnerabilities...
2023-04-24T14:52:05.847-0400	INFO	Number of language-specific files: 0

nginx:latest (debian 11.6)

Total: 32 (HIGH: 27, CRITICAL: 5)

┌──────────────┬────────────────┬──────────┬────────────────────┬───────────────┬──────────────────────────────────────────────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Installed Version  │ Fixed Version │                            Title                             │
├──────────────┼────────────────┼──────────┼────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ bash         │ CVE-2022-3715  │ HIGH     │ 5.1-2+deb11u1      │               │ bash: a heap-buffer-overflow in valid_parameter_transform    │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-3715                    │
├──────────────┼────────────────┼──────────┼────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ curl         │ CVE-2023-23914 │ CRITICAL │ 7.74.0-1.3+deb11u7 │               │ HSTS ignored on multiple requests                            │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-23914                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27536 │          │                    │               │ curl: GSS delegation too eager connection re-use             │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27536                   │
│              ├────────────────┼──────────┤                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2022-42916 │ HIGH     │                    │               │ curl: HSTS bypass via IDN                                    │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-42916                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2022-43551 │          │                    │               │ curl: HSTS bypass via IDN                                    │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-43551                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27533 │          │                    │               │ curl: TELNET option IAC injection                            │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27533                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27534 │          │                    │               │ curl: SFTP path ~ resolving discrepancy                      │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27534                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27535 │          │                    │               │ FTP too eager connection reuse                               │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27535                   │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ e2fsprogs    │ CVE-2022-1304  │          │ 1.46.2-2           │               │ e2fsprogs: out-of-bounds read/write via crafted filesystem   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-1304                    │
├──────────────┤                │          │                    ├───────────────┤                                                              │
│ libcom-err2  │                │          │                    │               │                                                              │
│              │                │          │                    │               │                                                              │
├──────────────┼────────────────┼──────────┼────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libcurl4     │ CVE-2023-23914 │ CRITICAL │ 7.74.0-1.3+deb11u7 │               │ HSTS ignored on multiple requests                            │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-23914                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27536 │          │                    │               │ curl: GSS delegation too eager connection re-use             │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27536                   │
│              ├────────────────┼──────────┤                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2022-42916 │ HIGH     │                    │               │ curl: HSTS bypass via IDN                                    │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-42916                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2022-43551 │          │                    │               │ curl: HSTS bypass via IDN                                    │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-43551                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27533 │          │                    │               │ curl: TELNET option IAC injection                            │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27533                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27534 │          │                    │               │ curl: SFTP path ~ resolving discrepancy                      │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27534                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2023-27535 │          │                    │               │ FTP too eager connection reuse                               │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-27535                   │
├──────────────┼────────────────┼──────────┼────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libdb5.3     │ CVE-2019-8457  │ CRITICAL │ 5.3.28+dfsg1-0.8   │               │ sqlite: heap out-of-bound read in function rtreenode()       │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2019-8457                    │
├──────────────┼────────────────┼──────────┼────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libext2fs2   │ CVE-2022-1304  │ HIGH     │ 1.46.2-2           │               │ e2fsprogs: out-of-bounds read/write via crafted filesystem   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-1304                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libgcrypt20  │ CVE-2021-33560 │          │ 1.8.7-6            │               │ libgcrypt: mishandles ElGamal encryption because it lacks    │
│              │                │          │                    │               │ exponent blinding to address a...                            │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2021-33560                   │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libss2       │ CVE-2022-1304  │          │ 1.46.2-2           │               │ e2fsprogs: out-of-bounds read/write via crafted filesystem   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-1304                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libssl1.1    │ CVE-2023-0464  │          │ 1.1.1n-0+deb11u4   │               │ Denial of service by excessive resource usage in verifying   │
│              │                │          │                    │               │ X509 policy constraints...                                   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-0464                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libtinfo6    │ CVE-2022-29458 │          │ 6.2+20201114-2     │               │ ncurses: segfaulting OOB read                                │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-29458                   │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libxpm4      │ CVE-2022-44617 │          │ 1:3.5.12-1         │               │ libXpm: Runaway loop on width of 0 and enormous height       │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-44617                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2022-46285 │          │                    │               │ libXpm: Infinite loop on unclosed comments                   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-46285                   │
│              ├────────────────┤          │                    ├───────────────┼──────────────────────────────────────────────────────────────┤
│              │ CVE-2022-4883  │          │                    │               │ libXpm: compression commands depend on $PATH                 │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-4883                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ libzstd1     │ CVE-2022-4899  │          │ 1.4.8+dfsg-2.1     │               │ buffer overrun in util.c                                     │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-4899                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ logsave      │ CVE-2022-1304  │          │ 1.46.2-2           │               │ e2fsprogs: out-of-bounds read/write via crafted filesystem   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-1304                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ ncurses-base │ CVE-2022-29458 │          │ 6.2+20201114-2     │               │ ncurses: segfaulting OOB read                                │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2022-29458                   │
├──────────────┤                │          │                    ├───────────────┤                                                              │
│ ncurses-bin  │                │          │                    │               │                                                              │
│              │                │          │                    │               │                                                              │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ openssl      │ CVE-2023-0464  │          │ 1.1.1n-0+deb11u4   │               │ Denial of service by excessive resource usage in verifying   │
│              │                │          │                    │               │ X509 policy constraints...                                   │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2023-0464                    │
├──────────────┼────────────────┤          ├────────────────────┼───────────────┼──────────────────────────────────────────────────────────────┤
│ perl-base    │ CVE-2020-16156 │          │ 5.32.1-4+deb11u2   │               │ perl-CPAN: Bypass of verification of signatures in CHECKSUMS │
│              │                │          │                    │               │ files                                                        │
│              │                │          │                    │               │ https://avd.aquasec.com/nvd/cve-2020-16156                   │
└──────────────┴────────────────┴──────────┴────────────────────┴───────────────┴──────────────────────────────────────────────────────────────┘
```
</details>

## Заключение

Запуск веб-сервера Nginx и базы данных Redis в контейнерах Docker позволяет быстро и удобно создавать окружения для разработки и тестирования. Docker обеспечивает изолированную среду, в которой можно запускать приложения и сервис

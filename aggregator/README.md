Ця папка містить все необхідне для розгортання агрегатора в продакшені та тестування існуючого функціоналу

### Вимоги

1. Встановити [Docker](https://docs.docker.com/engine/install/)
1. Встановити [Docker Compose V2 Plugin](https://docs.docker.com/compose/cli-command/#installing-compose-v2)

### Авторизація для доступу до докер образів

Образи `ghcr.io/sprmt/sprmt-aggregator` та `ghcr.io/sprmt/maps` є приватними. Для того щоб мати до них доступ вам потрібно:

1.  Мати доступ до образів. (На даний момент доступ наслідуються від доступу до репозитаріїв).
1.  Згенерувати персональний токен доступу (https://github.com/settings/tokens). Потрібно вибирати scopes `read:packages`, якщо потрібно тільки доступ до образів.
1.  Використовуючи цей токен як пароль, залогінитися в github registry. Приклад:

```bash
echo <TOKEN> | docker login ghcr.io -u <USERNAME> --password-stdin
```

Більше деталей: https://docs.github.com/en/github-ae@latest/packages/working-with-a-github-packages-registry/working-with-the-docker-registry

### Розгортання з нуля

1. Скопіювати зміст папки deployment на сервер. Якщо потрібно розгортати декілько інстансів агрегатора на одному сервері, то можна робити папки з різними назвами. При множинній інсталяції потрібно забезпечити щоб у кожної інсталяції був виділений свій IP (рекомендований варіант) Для цього потрібно тільки задати змінну `HOST_IP`.

   Або якщо такої можливості нема, і все на одному IP, то потрібно задавати унікальні порти для кожної інсталяції (змінні `HTTP_FE_PORT`, `MQTT_TCP_PORT`).

1. Відредагувати `.env` при необхідності.
   Важливі змінні:

   - `SPRMT_FRONTEND_CESIUM_ACCESS_TOKEN` Токен можно отримати безкоштовно на сайті https://ion.cesium.com/signin . Наявність токена дозволяє отримувати деякі додаткові мапи та більшу кількість запитів.
   - `SPRMT_IMAGE_TAG` який тег використовувати для агрегатору.
   - `HOST_IP` на яком IP і інтерфейсі буде працювати агрегатор. Можна прив'язувати на різні ВПН інтерфейси.
   - `HTTP_FE_PORT` порт на якому буде працювати агрегатор.

1. Підняти сервіси:

   ```bash
   docker compose up -d
   ```

### Оновлення

1. Відредагувати `.env`. Встановивши в змінній `SPRMT_IMAGE_TAG` новий тег. Список тегів можна подивитися в [списку іміджів](https://github.com/SPRMT/sprmt-aggregator/pkgs/container/sprmt-aggregator). Бранч `dev` має 2 тега: `dev` та `latest`. При кожному коміті в PR запускається білд іміджа з мердж комміта (рузальтат мерджа бранча PR та `dev`). Імідж отримує тег `pr-<PR number>`, наприклад: `pr-966`. Тобто для того щоб оновити імідж зібраний з PR потрібно встановити `SPRMT_IMAGE_TAG=pr-966`
1. Оновити зміст папки з репозиторія на сервері.
1. Підтягнути потрібні іміджи:

   ```bash
   docker compose pull
   ```

 1. Перестворити інстанси:
    Визначитися чи нова версія потребує нової структури бази даних. Це можна запитати у розробників.
    На даний момент міграція не реалізована, тому у випадку потреби нової версії бази даних, агрегатор не буде працювати. В цьому випадку потрібно буде перед оновленням зупинити сервіс та видалити volume з базою даних.

    1. Перестворити інстанси без видалення тома з базою даних:

    ```bash
    docker compose up -d --force-recreate
    ```
    :exclamation::exclamation::exclamation: Перестворення з видаленням призведе до ВИДАЛЕННЯ всіх даних з агрегатора!

    1. Перестворити інстанси з видаленням томів:
    ```bash
    docker compose down -v
    docker compose up -d
    ```

1. Перевірити що всі сервіси працюють:
    ```
    docker compose ps
    NAME                       IMAGE                                COMMAND                  SERVICE      CREATED          STATUS                    PORTS
    aggregator-aggregator-1    ghcr.io/sprmt/sprmt-aggregator:dev-binary   "/home/aggregator/sp…"   aggregator    30 minutes ago   Up 30 minutes (healthy)   
    aggregator-autoheal-1      willfarrell/autoheal                        "/docker-entrypoint …"   autoheal      30 minutes ago   Up 30 minutes (healthy)   
    aggregator-mqtt_broker-1   eclipse-mosquitto:2.0                       "/docker-entrypoint.…"   mqtt_broker   30 minutes ago   Up 30 minutes (healthy)   1883/tcp
    aggregator-web-1           nginx                                       "/docker-entrypoint.…"   web           30 minutes ago   Up 29 minutes (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:1883->1883/tcp
    ```

    Звернути увагу що всі сервіси повинні бути `(healthy)`

1. При необхідності перевірити логи від сервісу агрегатора:
   ```bash
   docker compose logs -f aggregator
   ```

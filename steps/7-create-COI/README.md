# Шаг 7. Создание экземпляра COI
## Создание лог-группы

Чтобы создать лог-группу `yc-logging`, выполните команду:

    yc logging group create \
    --name=yc-logging
    
    yc logging group list

Предыдущая команда покажет, что у вас имеются две лог-группы: `default` и `yc-logging`. В первую до этого момента записывались все логи, а во вторую мы направим логи из `COI`, для этого запомним `ID` новой группы.

## Создание экземпляра COI

В директории вы найдете файл `spec.yaml`, который нужно изменить:
1. Добавить после `image:` URL_IMAGE, полученный на предыдущих шагах.
2. Добавить после `YC_GROUP_ID:` ранее полученный `ID` лог-группы `yc-logging`.

В директории вы найдете файл `user-data.yaml`, в котором необходимо изменить секцию `users` (она используется для передачи в экземпляр `COI` данных о пользователях). В нашем случае два пользователя: `avhaliullin` и `golodnyj`, у каждого указан открытый SSH-ключ (ssh-rsa и ssh-ed25519). Поправьте эту секцию, чтобы вместо текущих пользователей был указан ваш пользователь с вашим ключом, получить который вы можете с помощью команды:

    pbcopy < ~/.ssh/id_ed25519.pub

Для создания экземпляра COI необходимо получить настройки сети:

    yc vpc subnet list

Планируя разместить экземпляр COI в зоне `ru-central1-a`, возьмите `NAME` подсети расположенной в этой зоне, (в моём случае это `network-for-15-ru-central1-a`). Возьмите COI-образ и создайте экземпляр с именем `coi-vm`:

    echo "export IMAGE_ID=$(yc compute image get-latest-from-family container-optimized-image --folder-id standard-images --format=json | jq -r .id)" >> ~/.bashrc && . ~/.bashrc
    echo $IMAGE_ID

    yc compute instance create \
    --name coi-vm \
    --zone=ru-central1-a \
    --network-interface subnet-name=network-for-15-ru-central1-a,nat-ip-version=ipv4 \
    --metadata-from-file user-data=user-data.yaml,docker-compose=spec.yaml \
    --create-boot-disk image-id=$IMAGE_ID \
    --service-account-id $SERVICE_ACCOUNT_ID

После того как экземпляр будет создан, можно получить `EXTERNAL_IP` виртуальной машины с `NAME`, равным `coi-vm`, и использовать его для проверки работы:

    yc compute instance list

    echo "export ENDPOINT=http://EXTERNAL_IP" >> ~/.bashrc && . ~/.bashrc
    echo $ENDPOINT

    curl -v ${ENDPOINT}/ && echo
    curl -v ${ENDPOINT}/user-error && echo
    curl -v ${ENDPOINT}/internal-error && echo
    curl -v ${ENDPOINT}/not-found && echo

Каждый из вызовов `curl` формирует отдельный запрос, который создаёт соответствующие записи в `Cloud Logging` в отдельной лог-группе с именем `yc-logging`. Если все запросы отработали, можно смело посмотреть результат их работы:

    yc logging read --group-name=yc-logging --format=json

## Видео

https://youtu.be/eOjBpGdviak
# Infrastructure
Серверная инфраструктура проекта CyberDAS, включающая систему мониторинга, логгирования и маршрутизации запросов.

## Что это?

Это набор конфигов и `docker-compose` файлов, настраивающих окружение на сервере проекта CyberDAS. Всё окружение состоит из набора обособленных контейнеров, выполняющих свою задачу. Если вы никогда не сталкивались с `docker` и вам не знакомо понятие контейнера, рекомендую пройти по этой ссылке: https://docs.docker.com/get-started/.

## Маршрутизация

В нашей инфраструктуре есть только один контейнер,'смотрящий' в публичную сеть и направляющий запросы к контейнерам - [traefik](https://github.com/traefik/traefik). 

Такое решение может показаться странным, ведь если упадет этот контейнер - то сразу теряется доступ ко всей остальной системе. Ну да. Но вариант поставить один глобальный `Nginx` был бы ни чем не лучше. 

Но у `traefik` есть один огромный плюс: добавление новых контейнеров производится без дописывания глобального конфига. Если разработчик хочет опубликовать новый сервис, он просто присоединяет свой контейнер к определенной `Docker network` (в нашем случае она называется "proxy") и объявляет в одной строчке о том, что ему нужен доступ в веб. Вуаля! `Docker` делает всю остальную работу. 

На примере (фрагмент `docker-compose` файла абстрактного сервиса):
```yaml
services:
  backend:
    build: backend
    expose:
      - 5000
    networks:
      - proxy
    labels:
      - traefik.enable=true

networks:
  proxy:
    external: true
```

Что при этом произойдет? 
1. Приложение, которое работает в контейнере `backend` на 5000-ом порту окажется доступным по адресу https://backend.cyberdas.net
2. Все HTTP запросы к контейнеру автоматически перенаправляются на протокол HTTPS
3. На этот поддомен автоматически оформится SSL-сертификат и будет обновляться, до тех пор пока этот контейнер существует
4. В логах доступа автоматически появится новая категория, позволяющая анализировать запросы только к этому контейнеру (ура, можно не вести своих access логов!)


Пара важных моментов:
1. При отсутствии ключа `TRAEFIK_REAL_CERTS` в окружении, `traefik` будет обращаться к стейджинг АПИ LetsEncrypt (отлично подходит для тестов!). Если этот ключ есть в окружении, то `traefik` будет пытаться получить настоящие сертификаты.
2. Раздача имен сервисам (`backend.cyberdas.net` из примера выше) происходит нехитрым образом: из полного имени контейнера, состоящего из имени самого контейнера и имени проекта, отрываемся имя проекта. То, что осталось - создается как сабдомен. **Следите за коллизиями и именуйте семантически!** Кстати, не все сервисы должны стоять на сабдоменах, и это поведение полностью кастомизируется. По всем вопросам обращайтесь к документации: https://doc.traefik.io/
3. При локальной разработке получить доступ к сервисам можно обращаясь на локалхост; например, `traefik.localhost`.

## Логирование

Логирование происходит в несколько этапов:
1. Логи формируются в приложении, будь то traefik или БД
2. Они отправляются в stdout/stderr, где их может читать Docker
3. Docker, используя драйвер `fluentd` отправляет их по указанному адресу (в нашем случае localhost:24224)
4. [Fluentd](https://www.fluentd.org/) производит первичную обработку и маршрутизацию логов, отправляя их [Loki](https://grafana.com/oss/loki/)
5. [Loki](https://grafana.com/oss/loki/) хранит логи до востребования

Такая сложная последовательность действий - не моя прихоть. 

Например, у Loki есть свой драйвер для Docker'а, но у него не хватает возможностей для первичной обработки. Без неё неотформатированные логи (например, от сторонних приложений, таких как traefik) очень сложно и долго обрабатываются в конечных системах.

А прослойка в виде Docker'а для отправки в Fluentd позволяет в любой момент поменять агента не внося правок в код приложений.

Главное не забыть что все компоненты этой системы должны видеть друг друга и правильно настроить сети.

И так, приняв такую тяжелую архитектуру, нужно понять как с ней работать:
1. Fluentd крутится на локалхосте (127.0.0.1:24244), потому что он должен быть доступен не только из одного `docker-compose`. Благодяра этому, в любых контейнерах в пределах одной машины мы можем сделать так:
```yaml
services:
  backend:
    build: backend
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: app.backend
```
2. Тэги - это важно. Вся маршрутизация и обработка логов в Fluentd основывается на тэгах. Сам конфиг лежит в `fluentd/fluent.conf`. В этом проекте такое соглашение: все самописные приложения имеют первоуровневым тэгом `app` и обязаны предоставлять структурированные JSON-логи, обязательно указывая `level` и поле `app_name`, которое в идеале совпадает с именем контейнера (не забываем про коллизии!). Сторонние приложения - тяжелый случай, так как формат их логов зачастую неконтроллируемый, поэтому для них можно, и даже нужно, делать кастомные тэги и писать свои обработчики.
3. Не нужно писать свои access логи, так как traefik предоставляет очень богатый набор данных, которого хватит с головой. Только в будущем нужно трэйсинг добавить. 

В случае возникновения вопросов обращайтесь к документации: https://docs.fluentd.org/
# TRASSIR Server Monitoring Template для Zabbix

Шаблон Zabbix 7.4 для мониторинга серверов и каналов TRASSIR через SDK API.

## Что мониторит шаблон

### Базовые элементы данных

- `trassir.health` (HTTP Agent, 2m) - состояние сервера TRASSIR
- `trassir.objects` (HTTP Agent, 12h) - список объектов TRASSIR
- `trassir.channels.online` (Dependent) - количество каналов онлайн
- `trassir.channels.total` (Dependent) - общее количество каналов
- `trassir.cpu.load` (Dependent, float, %) - загрузка CPU

### LLD: каналы

Правило `trassir.channel.discovery` обнаруживает объекты класса `Channel` и создает:

- `trassir.channel.signal[{#CHANNEL.GUID}]` (HTTP Agent, 2m)

Преобразование значения сигнала:

- `Signal` -> `1`
- любое другое/ошибка -> `0`

### LLD: удаленные серверы

Правило `trassir.server.discovery` обнаруживает объекты класса `Server` и создает:

- `trassir.server.connection[{#SERVER.GUID}]` (HTTP Agent, 2m)

Текущий хост (по `{HOST.HOST}`) исключается фильтром из обнаружения.

## Триггеры

### На уровне сервера

- `TRASSIR: Нет данных от регистратора (>2м)` - нет значений `trassir.health` более 121 секунды (HIGH)
- `TRASSIR: Критическая загрузка CPU (>90% за 30м)` - `avg(trassir.cpu.load,30m) >= 90` (HIGH)

### На уровне канала

- `TRASSIR: Канал [{#CHANNEL.NAME}]: Нет сигнала` - последнее значение сигнала равно `0` (WARNING)
- `TRASSIR: Канал [{#CHANNEL.NAME}]: Потеря сигнала ≥3 раз за 1ч` - `changecount(...,"dec") >= 3` (AVERAGE)
- `TRASSIR: Канал [{#CHANNEL.NAME}]: Потеря сигнала ≥8 раз за 1ч` - `changecount(...,"dec") >= 8` (HIGH)

Для триггеров потери сигнала используется recovery expression: проблема закрывается после 24 часов без обрывов и при наличии данных за последний час.

### На уровне удаленных серверов

- `TRASSIR: Регистратор [{#SERVER.NAME}]: Нет связи` - последнее значение подключения равно `0` (HIGH)
- `TRASSIR: Регистратор [{#SERVER.NAME}]: Нестабильная связь (24ч)` - были отключения за сутки (AVERAGE), зависит от триггера «Нет связи»

## Требования

- Zabbix 7.4+
- TRASSIR Server с включенным SDK API
- Сетевой доступ от Zabbix Server/Proxy к TRASSIR

## Установка

### 1. Импорт шаблона

1. Откройте Zabbix UI -> Configuration -> Templates
2. Нажмите Import
3. Выберите файл `trassir_template.yaml`
4. Подтвердите импорт

### 2. Привязка к хосту

1. Перейдите в Configuration -> Hosts
2. Создайте или откройте нужный хост
3. Привяжите шаблон `TRASSIR Server`

### 3. Настройка макросов

| Макрос | Описание | Значение по умолчанию |
|---|---|---|
| `{$TRASSIR.URL}` | URL TRASSIR SDK API | `https://192.168.1.200:8080` |
| `{$TRASSIR.USERNAME}` | Имя пользователя TRASSIR | `Admin` |
| `{$TRASSIR.PASSWORD}` | Пароль пользователя TRASSIR | `password` |

## Используемые API-запросы

Все запросы выполняются методом `POST`:

- `{$TRASSIR.URL}/health`
- `{$TRASSIR.URL}/objects/`
- `{$TRASSIR.URL}/objects/{guid}`

Тело запроса:

```text
username={$TRASSIR.USERNAME}&password={$TRASSIR.PASSWORD}
```

## Карты значений

- `Signal status`: `0 -> No Signal`, `1 -> Signal`
- `Connection status`: `0 -> Offline`, `1 -> Online`
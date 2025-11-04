# Send_to_FROST
# Отправка данных на FROST Server и их структура

Для централизованного хранения и обработки климатических данных используется сервер **FROST** (Fraunhofer Open Source SensorThings Server), который реализует стандарт **OGC SensorThings API**. Этот стандарт обеспечивает единый формат обмена данными между сенсорами, приложениями и внешними сервисами.

В данном разделе описывается процесс настройки FROST для приёма данных с локальных метеостанций Ecowitt GW2000/GW3000 через промежуточный сервер (Home Assistant), а также порядок отправки и проверки данных с помощью инструмента **Postman**, который предоставляет удобный интерфейс для тестирования REST API.

## Системные требования

- **FROST Server**: Версия 2.0+ с поддержкой SensorThings API v1.1
- **Сеть**: Доступ к `<адрес_сервера_FROST>` на порту 8080
- **Home Assistant**: Версия 2024.1+ с правами на выполнение REST-команд
- **Память**: ≥512 МБ для обработки потоков данных

### Перечень использованного ПО и оборудования:
- **FROST Server** - сервер данных SensorThings API
- **Postman** - тестирование REST API
- **Home Assistant RESTful Command** - интеграция отправки данных
- **Личный ПК** для работы с Home Assistant и Postman
- **Метеостанции Ecowitt** - источники метеоданных

## Шаги по отправке данных с Home Assistant на локальный сервер FROST

Создание сущностей выполнялось через **Postman** – инструмент тестирования REST API, позволяющий вручную отправлять запросы к серверу без написания кода. Такой подход является максимально простым и наглядным на этапе отладки, хотя в дальнейшем его можно заменить на автоматическую интеграцию (например, через Home Assistant или Python-скрипты).

### Настройка Postman

**Как работает Postman**:

1. **Выбор метода запроса** (HTTP method):
   - `GET` – запросить данные у сервера (например, список последних измерений)
   - `POST` – отправить новые данные на сервер (например, показания датчика)
   - `PUT / PATCH` – изменить уже существующие данные
   - `DELETE` – удалить данные

2. **Формирование запроса**:
   - указать URL-адрес сервера (например, `http://frost-server:8080/FROST-Server/v1.1/Observations`)
   - добавить заголовки (headers), чтобы сервер понял формат (`Content-Type: application/json`)
   - в теле запроса (body) передаем данные в формате JSON

3. **Отправка запроса**:
   - Postman формирует HTTP-пакет и отправляет его по сети на сервер

4. **Получение ответа**:
   - Сервер возвращает результат – это может быть:
     - JSON с нужными данными
     - подтверждение, что данные записаны
     - сообщение об ошибке

5. **Отображение и анализ**:
   - Postman показывает ответ в удобной форме (JSON, таблица, текст, код состояния)

**Скачать Postman можно по ссылке**: https://www.postman.com/downloads/

## Добавление сущностей

### Thing

**Thing** – это «объект наблюдения», то есть любая сущность, с которой связаны сенсоры. В нашем случае это метеостанция (GW2000 или GW3000).

**Тип запроса и адрес**: 
```bash
POST http://<адрес_сервера>:8080/FROST-Server/v1.1/Things
```
**Headers**:
Content-Type: application/json

**Body**:
```bash
json
{
  "name": "Метеостанция GW2000",
  "description": "Wi-Fi шлюз Ecowitt GW2000, подключенный к Home Assistant",
  "properties": {
    "location": "Raspberry Pi 5, Москва"
  }
}
```
**Sensor**
Sensor описывает устройство измерения. Например, при наличии одной метеостанции, у неё могут быть десятки сенсоров (температура, влажность, давление, PM2.5 и т.д.). Каждый датчик фиксируется как отдельный Sensor.
Тип запроса и адрес:
```bash
POST http://<адрес_сервера>:8080/FROST-Server/v1.1/Sensors
```
Headers:
Content-Type: application/json
**Body:**
```bash
{
  "name": "Температурный датчик WH32",
  "description": "Датчик температуры и влажности воздуха Ecowitt",
  "encodingType": "application/pdf",
  "metadata": "https://www.ecowitt.com/weather_station/wh32"
}
```
ObservedProperty описывает само физическое явление, которое измеряется: температура, влажность, давление, скорость ветра. Оно абстрактное и не зависит от конкретного датчика.
Тип запроса и адрес:
```bash
POST http://<адрес_сервера>:8080/FROST-Server/v1.1/ObservedProperties
```
**Headers:**
```bash
Content-Type: application/json
```
**Body:**
```bash
{
  "name": "Температура воздуха",
  "definition": "http://dbpedia.org/page/Temperature",
  "description": "Измерение температуры воздуха в градусах Цельсия"
}
```
**Datastream**
Datastream — это связка Thing + Sensor + ObservedProperty. То есть «какая метеостанция каким датчиком измеряет какое свойство». Без Datastream невозможно передавать измерения.
Тип запроса и адрес:
```bash
POST http://<адрес_сервера>:8080/FROST-Server/v1.1/Datastreams
```
**Headers:**
```bash
Content-Type: application/json
```
**Body:**
```bash
{
  "name": "Поток температуры",
  "description": "Непрерывные измерения температуры воздуха",
  "observationType": "http://www.opengis.net/def/observationType/OGC-OM/2.0/OM_Measurement",
  "unitOfMeasurement": {
    "name": "Градусы Цельсия",
    "symbol": "°C",
    "definition": "http://unitsofmeasure.org/ucum.html#para-30"
  },
  "Thing": {"@iot.id": 1},
  "Sensor": {"@iot.id": 1},
  "ObservedProperty": {"@iot.id": 1}
}
```
**Observation (наблюдения)**
Данный запрос проверяет корректность работы сущности (Thing, Sensor, ObservedProperty, Datastream); соответствие структуры данных стандарту OGC SensorThings API; корректность приема и сохранения данных сервером FROST.
Тип запроса и адрес:
```bash
POST http://<адрес_сервера>:8080/FROST-Server/v1.1/Observations
```
**Headers:**
```bash
Content-Type: application/json
```
**Body:**
```bash
{
  "phenomenonTime": "2025-10-03T15:00:00Z",
  "result": 12.3,
  "Datastream": {"@iot.id": 1}
}
```
## Автоматизация отправки данных с Home Assistant на сервер FROST

После настройки базовых сущностей через Postman необходимо настроить автоматическую отправку данных с Home Assistant на сервер FROST.

### Настройка Home Assistant

#### 1. Установка File Editor

Для редактирования конфигурационных файлов Home Assistant необходимо установить дополнение File Editor:

1. Перейдите в **Настройки → Дополнения → Магазин дополнений**
2. Найдите и установите **File Editor**
3. Запустите дополнение через веб-интерфейс

#### 2. Настройка configuration.yaml

В файле `configuration.yaml` добавьте в конец следующую строку:

```yaml
rest_command: !include rest_command.yaml
automation: !include automations.yaml

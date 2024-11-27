# Техническое задание на реализацию бэкенда системы бронирования авиабилетов

## 1. Входные параметры

| Параметр | Тип параметра | Обязательность | Описание |
|----------|---------------|----------------|-----------|
| flightId | uuid | + | Идентификатор рейса |
| userId | uuid | + | Индентификатор пользователя |
| bookingId | uuid | - | Индентификатор бронирования |
| luggageInfo | string | - | Информация о багаже |

## 2. Авторизация

### a. По бизнесу
- Доступ к функции бронирования билетов предоставляется всем авторизованным пользователям
- Просмотр и управление бронированием доступно только владельцу билета
- Доступ к информации о рейсе предоставляется всем пользователям

### b. Техническая реализация
```sql
-- Проверка прав доступа к бронированию
SELECT id FROM tbl_bookings 
WHERE user_id = {current_user} 
AND id = {bookingId}

-- Проверка статуса рейса
SELECT status FROM tbl_flights 
WHERE id = {flightId} 
AND status = 'ACTIVE'
```

## 3. Передача данных при инициализации

```json
{
  "flightInfo": {
    "departureCity": "",
    "arrivalCity": "",
    "departureDate": "18.10.2024",
    "arrivalDate": "21.11.2024",
    "flightNumbers": ["16:30 SVO", "22:05 ASF"]
  },
  "price": 13272,
  "passengerInfo": {
    "firstName": "",
    "lastName": "",
    "middleName": "",
    "documentType": "Российский паспорт",
    "documentNumber": "",
    "birthDate": ""
  },
  "contactInfo": {
    "email": "",
    "phone": ""
  },
  "luggage": {
    "handLuggage": {
      "included": true,
      "quantity": 1,
      "weight": 10
    },
    "checkedBaggage": {
      "included": false,
      "quantity": 1,
      "weight": 20,
      "price": 6750
    }
  }
}
```

### Описание атрибутов

| Атрибут, уровень 1 | Уровень 2 | Тип | Название атрибута | Формирование на бэкенде | Обязательность |
|-------------------|-----------|-----|-------------------|------------------------|----------------|
| flightInfo | departureCity | string | Город вылета | SELECT city_name FROM tbl_cities WHERE id = (SELECT departure_city_id FROM tbl_flights WHERE id = {flightId}) | + |
| flightInfo | arrivalCity | string | Город прилета | SELECT city_name FROM tbl_cities WHERE id = (SELECT arrival_city_id FROM tbl_flights WHERE id = {flightId}) | + |
| flightInfo | departureDate | string | Дата вылета | TO_CHAR(departure_date, 'DD MMM') FROM tbl_flights WHERE id = {flightId} | + |
| flightInfo | arrivalDate | string | Дата прилета | TO_CHAR(arrival_date, 'DD MMM') FROM tbl_flights WHERE id = {flightId} | + |
| price | - | number | Стоимость билета | SELECT base_price FROM tbl_flights WHERE id = {flightId} | + |
| passengerInfo | firstName | string | Имя пассажира | - | + |
| passengerInfo | lastName | string | Фамилия пассажира | - | + |
| passengerInfo | documentType | string | Тип документа | - | + |
| contactInfo | email | string | Email | - | + |
| contactInfo | phone | string | Телефон | - | + |
| luggage | handLuggage | object | Ручная кладь | SELECT * FROM tbl_luggage_types WHERE type = 'HAND' | + |
| luggage | checkedBaggage | object | Багаж | SELECT * FROM tbl_luggage_types WHERE type = 'CHECKED' | - |

## 4. REST-запросы

### 4.1 Получение информации о рейсе

| Название | Получение деталей рейса |
|----------|------------------------|
| URL | /api/flights/{flightId} |
| Тип метода | GET |
| Проверка авторизации | Не требуется |
| Действия на бэкенде | SELECT * FROM tbl_flights WHERE id = {flightId} |
| Query parameters | - |
| Responses | 200 OK: {flight_details}<br>404 Not Found: {"message": "Рейс не найден"} |

### 4.2 Создание бронирования

| Название | Создание бронирования билета |
|----------|----------------------------|
| URL | /api/bookings |
| Тип метода | POST |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | INSERT INTO tbl_bookings (flight_id, passenger_data, contact_info, luggage_info) VALUES (...) |
| Body | {<br>"flightId": "uuid",<br>"passengerInfo": {...},<br>"contactInfo": {...},<br>"luggageInfo": {...}<br>} |
| Responses | 201 Created: {"bookingId": "uuid"}<br>400 Bad Request: {"message": "Ошибка валидации"}<br>409 Conflict: {"message": "Нет свободных мест"} |

### 4.3 Оплата бронирования

| Название | Оплата забронированного билета |
|----------|------------------------------|
| URL | /api/bookings/{bookingId}/payment |
| Тип метода | POST |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | 1. Проверка статуса бронирования<br>2. Создание платежа<br>3. Обновление статуса бронирования |
| Body | {<br>"paymentMethod": "CARD"/"SBP",<br>"paymentDetails": {...}<br>} |
| Responses | 200 OK: {"message": "Оплата прошла успешно"}<br>400 Bad Request: {"message": "Ошибка оплаты"}<br>404 Not Found: {"message": "Бронирование не найдено"} |

## 5. Модель данных

![Изображение ER диграммы](./Снимок%20экрана%202024-10-29%20144109.png)

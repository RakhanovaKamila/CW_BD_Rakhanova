# 🗄️ Контрольна робота з баз даних
## Проектування БД для системи управління рестораном

> **Дисципліна:** Бази даних  
> **Тема:** Проектування реляційної БД для ресторану  
> **СУБД:** PostgreSQL  

---

## 📋 Зміст

1. [Опис завдання](#опис-завдання)
2. [ER-діаграма](#er-діаграма)
3. [Схема бази даних](#схема-бази-даних)
4. [OLTP-запити](#oltp-запити)
5. [OLAP-запити](#olap-запити)
6. [Висновки](#висновки)

---

## Опис завдання

Спроектувати базу даних для ресторану, що забезпечує управління:
- замовленнями клієнтів
- меню
- бронюванням столиків
- виконанням замовлень

### Необхідні сутності

| Сутність | Атрибути |
|---|---|
| **Клієнти** | ім'я, телефон, email |
| **Столики** | номер столика, місткість, розташування |
| **Позиції меню** | назва, опис, ціна, категорія |
| **Замовлення** | час замовлення, статус |
| **Позиції замовлення** | позиції в кожному замовленні з кількістю |

---

## ER-діаграма

```
┌─────────────┐        ┌──────────────┐        ┌─────────────┐
│   Client    │        │ Reservation  │        │   Tables    │
│─────────────│        │──────────────│        │─────────────│
│ ClientID PK │───1──<─│ ClientID  FK │        │ TableID  PK │
│ FullName    │        │ ReservationID│─>──1───│ TableNumber │
│ Phone       │        │ TableID   FK │        │ Capacity    │
│ Email       │        │ ReservationTime│       │ Location    │
└─────────────┘        │ GuestCount   │        └─────────────┘
       │               └──────────────┘               │
       │                                              │
       │1             ┌──────────────┐               │1
       │              │    Order     │               │
       └────1──────<──│──────────────│──>─────1──────┘
                      │ OrderID   PK │
                      │ ClientID  FK │
                      │ TableID   FK │
                      │ OrderTime    │
                      │ Status       │
                      └──────┬───────┘
                             │1
                             │
                            <│>
                             │
                      ┌──────┴───────┐        ┌─────────────┐
                      │  OrderItem   │        │  MenuItem   │
                      │──────────────│        │─────────────│
                      │ OrderItemID  │─>──1───│ MenuItemID  │
                      │ OrderID   FK │        │ Name        │
                      │ ItemID    FK │        │ Description │
                      │ Quantity     │        │ Price       │
                      └──────────────┘        │ Category    │
                                              └─────────────┘
```

**Зв'язки:**
- `Client` → `Order` : **1:N** (клієнт може мати багато замовлень)
- `Tables` → `Order` : **1:N** (за столиком може бути багато замовлень)
- `Order` → `OrderItem` : **1:N** (замовлення містить багато позицій)
- `MenuItem` → `OrderItem` : **1:N** (позиція меню може бути в багатьох замовленнях)
- `Client` → `Reservation` : **1:N** (клієнт може мати кілька бронювань)
- `Tables` → `Reservation` : **1:N** (столик може бронюватися багато разів)

---

## Схема бази даних

### DDL — Створення таблиць

```sql
-- 1. Таблиця Клієнтів
CREATE TABLE Client (
    ClientID  INT          PRIMARY KEY,
    FullName  VARCHAR(150) NOT NULL,
    Phone     VARCHAR(20),
    Email     VARCHAR(100)
);

-- 2. Таблиця Столиків
CREATE TABLE Tables (
    TableID     INT         PRIMARY KEY,
    TableNumber INT         NOT NULL,
    Capacity    INT         NOT NULL,
    Location    VARCHAR(100)
);

-- 3. Таблиця Меню
CREATE TABLE MenuItem (
    MenuItemID  INT            PRIMARY KEY,
    Name        VARCHAR(150)   NOT NULL,
    Description TEXT,
    Price       DECIMAL(10, 2) NOT NULL,
    Category    VARCHAR(50)
);

-- 4. Таблиця Бронювань
CREATE TABLE Reservation (
    ReservationID   INT PRIMARY KEY,
    ClientID        INT,
    TableID         INT,
    ReservationTime TIMESTAMP,
    GuestCount      INT,
    FOREIGN KEY (ClientID) REFERENCES Client(ClientID),
    FOREIGN KEY (TableID)  REFERENCES Tables(TableID)
);

-- 5. Таблиця Замовлень
-- Примітка: ORDER є зарезервованим словом у SQL, тому використовуються подвійні лапки
CREATE TABLE "Order" (
    OrderID   INT          PRIMARY KEY,
    ClientID  INT,
    TableID   INT,
    OrderTime TIMESTAMP,
    Status    VARCHAR(50),
    FOREIGN KEY (ClientID) REFERENCES Client(ClientID),
    FOREIGN KEY (TableID)  REFERENCES Tables(TableID)
);

-- 6. Таблиця Позицій замовлення
CREATE TABLE OrderItem (
    OrderItemID INT PRIMARY KEY,
    OrderID     INT,
    ItemID      INT,
    Quantity    INT NOT NULL CHECK (Quantity > 0),
    FOREIGN KEY (OrderID) REFERENCES "Order"(OrderID),
    FOREIGN KEY (ItemID)  REFERENCES MenuItem(MenuItemID)
);
```

---

## OLTP-запити

### 1. INSERT — Додавання даних

```sql
-- Додавання клієнтів
INSERT INTO Client (ClientID, FullName, Phone, Email)
VALUES
    (1, 'Олександр Коваленко', '+380501112233', 'kovalenko@email.com'),
    (2, 'Марія Бондар',        '+380674445566', 'bondar.m@email.com'),
    (3, 'Дмитро Шевченко',     '+380937778899', 'shevchenko.d@email.com');

-- Додавання столиків
INSERT INTO Tables (TableID, TableNumber, Capacity, Location)
VALUES
    (1, 1, 2, 'Біля вікна'),
    (2, 2, 4, 'Основний зал'),
    (3, 3, 6, 'Тераса');

-- Додавання позицій меню
INSERT INTO MenuItem (MenuItemID, Name, Description, Price, Category)
VALUES
    (1, 'Борщ український',  'Зі свіжою сметаною та пампушками',            180.00, 'Перші страви'),
    (2, 'Стейк Рібай',       'Яловичина зернової відгодівлі, просмажка Medium', 550.00, 'Основні страви'),
    (3, 'Лимонад класичний', 'Лимонний сік, м''ята, газована вода',           85.00, 'Напої');

-- Додавання бронювань
INSERT INTO Reservation (ReservationID, ClientID, TableID, ReservationTime, GuestCount)
VALUES
    (1, 1, 3, '2026-04-01 19:00:00', 5),
    (2, 2, 1, '2026-04-02 18:30:00', 2);

-- Додавання замовлень
INSERT INTO "Order" (OrderID, ClientID, TableID, OrderTime, Status)
VALUES
    (1, 1, 3, '2026-04-01 19:15:00', 'В процесі'),
    (2, 3, 2, '2026-04-01 14:00:00', 'Виконано');

-- Додавання позицій замовлень
INSERT INTO OrderItem (OrderItemID, OrderID, ItemID, Quantity)
VALUES
    (1, 1, 1, 2),  -- 2 борщі для замовлення №1
    (2, 1, 2, 2),  -- 2 стейки для замовлення №1
    (3, 2, 3, 1),  -- 1 лимонад для замовлення №2
    (4, 2, 1, 1);  -- 1 борщ для замовлення №2
```

---

### 2. UPDATE — Оновлення даних

```sql
-- Оновлення статусу замовлення на "Виконано"
UPDATE "Order"
SET Status = 'Виконано'
WHERE OrderID = 1;

-- Підвищення цін на основні страви на 15%
UPDATE MenuItem
SET Price = Price * 1.15
WHERE Category = 'Основні страви';
```

---

### 3. DELETE — Видалення даних

```sql
-- Видалення бронювання
DELETE FROM Reservation
WHERE ReservationID = 2;

-- Видалення конкретної позиції зі замовлення
DELETE FROM OrderItem
WHERE OrderID = 1 AND ItemID = 2;
```

---

### 4. SELECT — Вибірка даних

#### 4.1 Підготовка чеку для замовлення

```sql
SELECT
    m.Name              AS "Назва страви",
    oi.Quantity         AS "Кількість",
    m.Price             AS "Ціна за одиницю",
    oi.Quantity * m.Price AS "Сума рядка"
FROM OrderItem oi
JOIN MenuItem m ON oi.ItemID = m.MenuItemID
WHERE oi.OrderID = 1;
```

**Приклад результату:**

| Назва страви | Кількість | Ціна за одиницю | Сума рядка |
|---|---|---|---|
| Борщ український | 2 | 180.00 | 360.00 |
| Стейк Рібай | 2 | 550.00 | 1100.00 |

---

#### 4.2 Знайти всі вільні столики

```sql
-- Знайти столики, за якими немає незавершених замовлень
SELECT TableID, TableNumber, Capacity, Location
FROM Tables
WHERE TableID NOT IN (
    SELECT TableID
    FROM "Order"
    WHERE Status = 'В процесі'
);
```

**Логіка:** Підзапит повертає `TableID` столиків, де є активні замовлення зі статусом `'В процесі'`. Зовнішній запит виключає їх із результату.

---

## OLAP-запити

### 5. Загальний денний дохід за останній місяць

```sql
SELECT
    DATE(o.OrderTime)           AS "Дата",
    SUM(oi.Quantity * m.Price)  AS "Денний дохід"
FROM "Order" o
JOIN OrderItem oi ON o.OrderID = oi.OrderID
JOIN MenuItem  m  ON oi.ItemID = m.MenuItemID
WHERE o.OrderTime >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(o.OrderTime)
ORDER BY "Дата" DESC;
```

**Опис:** Агрегує суму добутків `Кількість × Ціна` по кожній даті за останні 30 днів, сортуючи від найновішої до найстарішої.

---

### 6. Топ-10 найпопулярніших позицій меню

```sql
SELECT
    m.Name              AS "Назва страви",
    SUM(oi.Quantity)    AS "Замовлено порцій"
FROM OrderItem oi
JOIN MenuItem m ON oi.ItemID = m.MenuItemID
GROUP BY m.Name
ORDER BY "Замовлено порцій" DESC
LIMIT 10;
```

**Опис:** Підраховує загальну кількість замовлених порцій для кожної позиції меню та повертає 10 найпопулярніших.

---

### 7. Середній чек за часом доби

```sql
SELECT
    CASE
        WHEN EXTRACT(HOUR FROM OrderTime) BETWEEN 8  AND 11 THEN 'Сніданок'
        WHEN EXTRACT(HOUR FROM OrderTime) BETWEEN 12 AND 16 THEN 'Обід'
        ELSE 'Вечеря'
    END                             AS "Час доби",
    ROUND(AVG(TotalOrderSum), 2)    AS "Середній чек"
FROM (
    -- Підзапит: сума кожного замовлення
    SELECT
        o.OrderID,
        o.OrderTime,
        SUM(oi.Quantity * m.Price) AS TotalOrderSum
    FROM "Order" o
    JOIN OrderItem oi ON o.OrderID = oi.OrderID
    JOIN MenuItem  m  ON oi.ItemID = m.MenuItemID
    GROUP BY o.OrderID, o.OrderTime
) AS OrdersWithTotals
GROUP BY "Час доби";
```

**Логіка розбивки:**

| Час доби | Години |
|---|---|
| Сніданок | 08:00 – 11:59 |
| Обід | 12:00 – 16:59 |
| Вечеря | 17:00 – 23:59 (та нічні години) |

---

### 8. Клієнти з найбільшими витратами (CTE)

```sql
WITH CustomerSpending AS (
    SELECT
        c.FullName,
        c.Email,
        SUM(oi.Quantity * m.Price) AS TotalSpent
    FROM Client c
    JOIN "Order"   o  ON c.ClientID  = o.ClientID
    JOIN OrderItem oi ON o.OrderID   = oi.OrderID
    JOIN MenuItem  m  ON oi.ItemID   = m.MenuItemID
    GROUP BY c.ClientID, c.FullName, c.Email
)
SELECT
    FullName,
    TotalSpent
FROM CustomerSpending
ORDER BY TotalSpent DESC
LIMIT 5;  -- Топ-5 найвідданіших клієнтів
```

**Опис:** CTE `CustomerSpending` обчислює загальну суму витрат кожного клієнта по всіх його замовленнях. Фінальний SELECT сортує за спаданням витрат та виводить п'ятірку найкращих.

---

## Висновки

У ході виконання контрольної роботи було:

1. **Спроектовано реляційну БД** з 6 таблицями: `Client`, `Tables`, `MenuItem`, `Reservation`, `Order`, `OrderItem`
2. **Визначено зв'язки** між сутностями через зовнішні ключі (`FOREIGN KEY`)
3. **Реалізовано OLTP-запити** для повсякденних операцій (INSERT, UPDATE, DELETE, SELECT)
4. **Реалізовано OLAP-запити** для аналітики: денний дохід, популярність страв, середній чек за часом доби, рейтинг клієнтів за витратами з використанням CTE

Схема нормалізована до **3НФ**: відсутні надлишкові залежності, кожна таблиця описує одну сутність, усі неключові атрибути залежать лише від первинного ключа.

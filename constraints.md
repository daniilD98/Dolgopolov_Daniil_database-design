# Обоснование ограничений

Схема `dolgopolov_food`, PostgreSQL. Имена ограничений в таблицах ниже — те, что присвоила база (в скриптах они не заданы явно); их можно увидеть запросом к `pg_constraint`. Формат обоснования: «что ограничиваем — потому что BR-N.M требует что».

---

## Enum-типы

Все перечисления — `CREATE TYPE ... AS ENUM` (BR-11.5), значения и порядок как в бизнес-правилах, ничего не добавлено.

| Тип | Значения | BR |
|---|---|---|
| `restaurant_status` | `активен`, `неактивен` | BR-4.3 |
| `product_status` | `доступно`, `нет_в_наличии`, `архив` | BR-5.5 |
| `courier_status` | `свободен`, `занят`, `на_доставке`, `уволен` | BR-6.2 |
| `order_status` | `создан`, `готовится`, `в_доставке`, `доставлен`, `отменён` | BR-7.7 |
| `order_pay_status` | `ожидает`, `оплачен`, `ошибка` | BR-7.9 |
| `payment_status` | `успех`, `ошибка`, `в_обработке` | BR-8.2 |
| `payment_type` | `карта`, `наличные`, `безнал` | BR-8.2 |
| `review_entity_type` | `блюдо`, `ресторан`, `заказ` | BR-9.1 |

---

## Ловушки

### Ловушка 1: `payment` — неполная сущность

**Какие атрибуты добавил в `payment` и почему** (BR-8.2):

```
payment_type      payment_type   — тип оплаты (карта / наличные / безнал), BR-8.2
amount            numeric(9,2)   — сумма попытки, BR-8.2; равна total_amount заказа (BR-8.4)
transaction_id    uuid, UNIQUE   — внешний идентификатор транзакции от платёжного провайдера, BR-8.2, BR-8.3
payment_timestamp timestamp      — дата и время транзакции, BR-8.2, DEFAULT localtimestamp (BR-11.2)
status            payment_status — статус попытки (успех / ошибка / в_обработке), BR-8.2
```

`transaction_id` имеет тип `uuid`: BR-11.1 требует UUID для всех идентификаторов, BR-8.3 говорит только, что идентификатор внешний и уникальный. Номера карт не хранятся (BR-1.7, BR-8.5) — только идентификатор транзакции.

**Где хранится тип оплаты — `payment` или `order`** (BR-8.6):

```
В payment. BR-8.6 прямо относит тип оплаты к попытке, а не к заказу: у одного заказа могут быть
попытки с разными типами (карта не прошла — оплатили наличными). Если бы тип лежал в "order",
он зависел бы не от заказа, а от конкретной попытки, и несколько типов у одного заказа были бы невозможны.
В "order" хранится только агрегат pay_status (BR-7.8).
```

**Остальные правила оплаты:**

```
BR-8.1 «не более одной успешной попытки на заказ» — частичный уникальный индекс
  uq_payment_one_success ON payment (order_id) WHERE status = 'успех'.
  «Ровно одна» целиком схемой не гарантируется: попытки появляются по одной, и у заказа до оплаты
  успешной попытки ещё нет (BR-8.7 описывает и такие заказы: 'ожидает', 'ошибка').
BR-8.4 «сумма попытки = общая сумма заказа» — составной внешний ключ
  payment (order_id, amount) → "order" (order_id, total_amount); для него в "order" есть UNIQUE (order_id, total_amount).
BR-8.7 «агрегированный статус оплаты» — триггер trg_payment_sync_pay_status пересчитывает order.pay_status
  при добавлении, изменении статуса и удалении попытки:
  есть успешная -> 'оплачен'; иначе есть 'в_обработке' -> 'ожидает'; иначе, если попытки есть, -> 'ошибка'.
```

### Ловушка 2: `order.address_id` — принадлежность клиенту (BR-7.4)

**Как обеспечено, что адрес доставки принадлежит клиенту:**

```
Составной внешний ключ:
  FOREIGN KEY (client_id, address_id) REFERENCES client_address (client_id, address_id)   -- order_client_id_address_id_fkey
Таблица client_address хранит пары «клиент — адрес» (PK по обоим столбцам). Заказ можно оформить только на пару,
которая есть в этой таблице, поэтому чужой адрес отклоняется самой базой, без триггера.
Побочный эффект (нужный): привязку клиент–адрес нельзя удалить, пока по ней есть заказы (ON DELETE RESTRICT) — BR-3.5.
Отдельные внешние ключи order_client_id_fkey и order_address_id_fkey оставлены: это связи заказа с клиентом и адресом из erd.md.
```

### Ловушка 3: `order_structure` — один ресторан в заказе (BR-7.3)

**Как обеспечено, что все блюда из `order.restaurant_id`:**

```
Триггер trg_order_structure_check (BEFORE INSERT OR UPDATE OF order_id, product_id на order_structure):
блюдо должно принадлежать тому же ресторану, что и заказ; иначе RAISE EXCEPTION с номером правила.

Почему не составной внешний ключ: для него в order_structure пришлось бы хранить restaurant_id, а он определяется
одним order_id — частью составного PK. Это частичная зависимость, нарушение 2НФ. Триггер сохраняет 3НФ.

Ограничение решения: триггер проверяет добавление и изменение позиций заказа. Смену ресторана у уже существующего
заказа или блюда он не отслеживает (отдельных триггеров на это нет — они выходили за рамки задания).
```

### Ловушка 4: `review` — полиморфизм (BR-9.1)

**Как реализована валидность `review.entity_id`:**

```
Гибрид: пара (entity_type, entity_id) сохранена, как в erd.md, а корректность обеспечивается так:
1) Составной внешний ключ (order_id, client_id) → "order"(order_id, client_id) — review_order_id_client_id_fkey:
   отзыв оставляет тот клиент, который оформил этот заказ (BR-9.2). Для него в "order" есть UNIQUE (order_id, client_id).
2) Триггер trg_review_check (BEFORE INSERT OR UPDATE): 'блюдо' — блюдо должно быть в order_structure этого заказа;
   'ресторан' — entity_id равен order.restaurant_id; 'заказ' — entity_id равен order_id (BR-9.2: отзыв только на то, что заказано).
   При UPDATE менять заказ, тип или объект отзыва нельзя — отзыв остаётся привязан к своему объекту.
3) UNIQUE (order_id, entity_type, entity_id) — не более одного отзыва на объект в рамках заказа (BR-9.5).

Обычный FK на entity_id невозможен: один столбец ссылается на три разные таблицы. Отдельные таблицы или три nullable-столбца
с CHECK изменили бы структуру из erd.md и усложнили триггер пересчёта рейтинга rating_change (FOOD-105).
Осиротевших ссылок не возникает: удалить блюдо из заказа, заказ с отзывом или ресторан с заказами нельзя (RESTRICT).
```

---

## CHECK-ограничения

| Таблица | Столбец | Условие (имя в БД) | BR | Обоснование |
|---|---|---|---|---|
| `restaurant` | `rating` | `rating BETWEEN 0 AND 5` (`restaurant_rating_check`) | BR-4.2 | BR-4.2 задаёт диапазон рейтинга ресторана; `NULL` проверку проходит — рейтинга ещё нет |
| `restaurant` | `work_start`, `work_end` | `work_end > work_start` (`restaurant_check`) | BR-4.4 | время закрытия позже времени открытия |
| `product` | `price` | `price >= 0` (`product_price_check`) | BR-5.3 | цена неотрицательна |
| `product` | `rating` | `rating BETWEEN 0 AND 5` (`product_rating_check`) | BR-5.4 | диапазон рейтинга блюда |
| `courier` | `coefficient` | `coefficient BETWEEN 0.1 AND 0.5` (`courier_coefficient_check`) | BR-6.4 | коэффициент курьера от 0.1 до 0.5 включительно |
| `courier` | `status`, `dismissal_date` | `status <> 'уволен' OR dismissal_date IS NOT NULL` (`courier_check`) | BR-6.3 | уволенный курьер обязан иметь дату увольнения; для работающего она не обязательна, поэтому обратное условие не накладывается |
| `courier_pay` | `amount` | `amount >= 0` (`courier_pay_amount_check`) | BR-10.4 | зарплата неотрицательна |
| `order` | `delivery_timestamp` | `delivery_timestamp IS NULL OR delivery_timestamp >= order_timestamp` (`order_check`) | BR-7.13 | доставка не может закончиться раньше создания заказа |
| `order` | `status`, `delivery_timestamp` | `status <> 'доставлен' OR delivery_timestamp IS NOT NULL` (`order_check1`) | BR-7.12 | время окончания доставки заполняется, когда заказ получает статус «доставлен»; до этого может быть пустым |
| `order_structure` | `quantity` | `quantity > 0` (`order_structure_quantity_check`) | BR-7.15 | количество положительное (может быть дробным) |
| `review` | `rating` | `rating BETWEEN 1 AND 5` (`review_rating_check`) | BR-9.4 | рейтинг отзыва от 1 до 5 |

### Правила, не вынесенные в ограничения БД

```
BR-7.11 (общая сумма заказа не менее 1000 ₽) и BR-1.8 (минимальная сумма заказа — 1000 ₽): в схеме отдельного CHECK нет.
  Условие соблюдается генератором insert_test_data (сумма заказа всегда не меньше 1000), но база сама
  заказ дешевле 1000 ₽ не отклонит.
BR-6.6 (в системе есть хотя бы один активный курьер): в самом правиле сказано, что это правило данных,
  проверяется в insert_test_data и на уровне схемы не гарантируется.
BR-8.1 («ровно одна» успешная попытка): гарантируется «не более одной» (см. ловушку 1).
```

---

## UNIQUE-ограничения

| Таблица | Столбцы (имя в БД) | BR | Обоснование |
|---|---|---|---|
| `client` | `login` (`client_login_key`) | BR-2.2 | логин уникален |
| `client` | `phone_number` (`client_phone_number_key`) | BR-2.2 | телефон уникален |
| `client_address` | PK `(client_id, address_id)` | BR-2.5 | один и тот же адрес нельзя привязать к одному клиенту дважды; разные клиенты могут делить адрес (BR-2.4) |
| `restaurant` | `address_id` (`restaurant_address_id_key`) | BR-3.3 | два ресторана не могут делить один физический адрес |
| `courier_pay` | PK `(year, month, courier_id)` | BR-10.3 | за один месяц у курьера ровно одна запись |
| `order_structure` | PK `(order_id, product_id)` | BR-7.16 | блюдо в заказе не повторяется — вместо этого растёт количество |
| `payment` | `transaction_id` (`payment_transaction_id_key`) | BR-8.3 | идентификатор транзакции от провайдера уникален |
| `payment` | частичный индекс `uq_payment_one_success`: `(order_id) WHERE status = 'успех'` | BR-8.1 | не более одной успешной попытки на заказ; неуспешных может быть сколько угодно |
| `review` | `(order_id, entity_type, entity_id)` (`review_order_id_entity_type_entity_id_key`) | BR-9.5 | не более одного отзыва на каждый объект в рамках заказа |
| `order` | `(order_id, client_id)` (`order_order_id_client_id_key`) | BR-9.2 | служебный: цель составного FK из `review`; PK уже уникален, ограничение позволяет сослаться на пару |
| `order` | `(order_id, total_amount)` (`order_order_id_total_amount_key`) | BR-8.4 | служебный: цель составного FK из `payment` (сумма попытки = общая сумма заказа) |

---

## FK-ограничения

Правило удаления: `RESTRICT` там, где иначе пропала бы история (BR-3.5). `CASCADE` — только у зависимых строк, которые не имеют смысла без родителя: `client_address → client` и `order_structure → order`.

| Таблица | Столбцы (имя в БД) | → Таблица | BR | Обоснование |
|---|---|---|---|---|
| `client_address` | `client_id` (`client_address_client_id_fkey`), CASCADE | `client` | BR-2.4 | у клиента несколько адресов; привязка не нужна без клиента |
| `client_address` | `address_id` (`client_address_address_id_fkey`), RESTRICT | `address` | BR-2.4, BR-3.5 | адрес не удаляется, пока на него есть ссылки |
| `restaurant` | `address_id` (`restaurant_address_id_fkey`) | `address` | BR-3.3, BR-3.5 | у ресторана ровно один адрес |
| `product` | `product_category_id` (`product_product_category_id_fkey`) | `product_category` | BR-5.1 | блюдо принадлежит ровно одной категории |
| `product` | `restaurant_id` (`product_restaurant_id_fkey`) | `restaurant` | BR-5.1 | блюдо принадлежит ровно одному ресторану |
| `courier_pay` | `courier_id` (`courier_pay_courier_id_fkey`) | `courier` | BR-10.2 | выплата относится к курьеру |
| `order` | `courier_id` (`order_courier_id_fkey`) | `courier` | BR-7.5 | доставку выполняет курьер |
| `order` | `client_id` (`order_client_id_fkey`) | `client` | BR-7.1 | заказ оформляет клиент |
| `order` | `restaurant_id` (`order_restaurant_id_fkey`) | `restaurant` | BR-7.3 | заказ оформлен в ресторане |
| `order` | `address_id` (`order_address_id_fkey`) | `address` | BR-7.4 | заказ доставляется по адресу |
| `order` | `(client_id, address_id)` (`order_client_id_address_id_fkey`) | `client_address` | BR-7.4 | адрес доставки — один из адресов именно этого клиента (ловушка 2) |
| `order_structure` | `order_id` (`order_structure_order_id_fkey`), CASCADE | `order` | BR-7.14 | позиция принадлежит заказу |
| `order_structure` | `product_id` (`order_structure_product_id_fkey`) | `product` | BR-7.14 | позиция — это блюдо и количество |
| `payment` | `order_id` (`payment_order_id_fkey`) | `order` | BR-8.1 | попытка оплаты относится к заказу |
| `payment` | `(order_id, amount)` (`payment_order_id_amount_fkey`) | `order (order_id, total_amount)` | BR-8.4 | сумма попытки равна общей сумме заказа |
| `review` | `client_id` (`review_client_id_fkey`) | `client` | BR-9.1 | отзыв оставляет клиент |
| `review` | `order_id` (`review_order_id_fkey`) | `order` | BR-9.2 | отзыв всегда связан с конкретным заказом |
| `review` | `(order_id, client_id)` (`review_order_id_client_id_fkey`) | `order (order_id, client_id)` | BR-9.2 | отзыв оставляет клиент, оформивший этот заказ |

Простые внешние ключи на `client`, `address` и `order` оставлены рядом с составными: они показывают все связи из erd.md.

---

## NOT NULL (только бизнес-обязательные)

| Таблица | Столбец | BR | Обоснование |
|---|---|---|---|
| `order` | `courier_id` | BR-7.5 | доставку выполняет курьер: заказ без курьера не существует |
| `restaurant` | `description`, `inn`, `ogrn`, `settlement_account` | BR-4.1, BR-4.5 | у ресторана есть описание и реквизиты (ИНН, ОГРН, расчётный счёт) |
| `courier` | `last_name`, `first_name` | BR-6.1 | у курьера есть ФИО; отчество — отдельное решение (см. раздел NULL) |
| `order` | `commission`, `total_amount` | BR-7.10 | вычисляемые столбцы всегда имеют значение |
| `payment` | все столбцы | BR-8.2 | у каждой попытки есть тип, сумма, идентификатор транзакции, время и статус |
| `review` | `rating` | BR-9.3 | у отзыва есть рейтинг и дата |

---

## NULL (осознанно разрешено)

| Таблица | Столбец | BR | Почему NULL |
|---|---|---|---|
| `address` | `apartment_office` | BR-3.2, BR-11.3 | у частных домов и отдельных входов указывать нечего |
| `product` | `photo_url` | BR-5.6, BR-11.3 | фотография необязательна |
| `review` | `photo_url` | BR-9.3, BR-11.3 | фотография отзыва необязательна |
| `courier` | `dismissal_date` | BR-6.3, BR-11.3 | у работающего курьера даты увольнения нет |
| `order` | `delivery_timestamp` | BR-7.12, BR-11.3 | до доставки время окончания неизвестно |
| `restaurant`, `product` | `rating` | BR-9.7 | агрегат отзывов: пока отзывов нет, рейтинга нет |
| `order` | `description` | BR-7.10 | комментарий к заказу не входит в обязательные атрибуты заказа и бывает пустым |
| `courier` | `patronymic` | BR-6.1 | отчество есть не у каждого человека, поэтому обязательным его не делаем |
| `review` | `description` | BR-9.3 | допущение: оценку можно поставить без текста. BR-9.3 перечисляет текст среди атрибутов отзыва и необязательной называет только фотографию, поэтому это решение вынесено на подтверждение |

---

## DEFAULT и GENERATED

| Таблица | Столбец | Что | Обоснование |
|---|---|---|---|
| все | `*_id` (PK) | `DEFAULT exts.uuid_generate_v4()` | BR-11.1: ID генерируются на сервере через расширение схемы `exts` |
| `courier` | `coefficient` | `DEFAULT 0.1` | BR-6.5: коэффициент нового курьера — 0.1 |
| `order` | `commission` | `GENERATED ALWAYS AS (round(order_amount * 0.10, 2)) STORED` | BR-1.4, BR-7.10: комиссия — ровно 10% от суммы без комиссии. Вычисляемый столбец не может разойтись с суммой заказа, в отличие от обычного поля с CHECK, которое пришлось бы заполнять вручную |
| `order` | `total_amount` | `GENERATED ALWAYS AS (order_amount + round(order_amount * 0.10, 2)) STORED` | BR-7.10: общая сумма = сумма без комиссии + комиссия. Комиссия в выражении повторена, потому что вычисляемый столбец не может ссылаться на другой вычисляемый |
| `order` | `order_timestamp` | `DEFAULT localtimestamp` | BR-7.1, BR-11.2: время создания в часовом поясе сервера |
| `order` | `status`, `pay_status` | `DEFAULT 'создан'`, `'ожидает'` | новый заказ создан, попыток оплаты ещё нет (BR-8.7) |
| `payment`, `review` | `payment_timestamp`, `review_timestamp` | `DEFAULT localtimestamp` | BR-8.2, BR-9.3, BR-11.2 |

### Размерность `numeric` и `varchar` (BR-11.6)

Размеры подобраны по реалистичным максимумам, а не «с запасом»:

| Столбец | Тип | Что вмещает |
|---|---|---|
| `product.price` | `numeric(7,2)` | цена блюда до 99 999.99 ₽ |
| `order.order_amount` | `numeric(8,2)` | стоимость блюд в заказе до 999 999.99 ₽ |
| `order.commission` | `numeric(7,2)` | 10% от суммы заказа, до 99 999.99 ₽ |
| `order.total_amount`, `payment.amount` | `numeric(9,2)` | сумма с комиссией, до 1 099 999.99 ₽; тип `payment.amount` совпадает с `total_amount`, потому что они связаны внешним ключом |
| `courier_pay.amount` | `numeric(8,2)` | зарплата за месяц до 999 999.99 ₽ |
| `order_structure.quantity` | `numeric(5,2)` | до 999.99 порций (дробные допустимы — BR-7.15) |
| `courier.coefficient`, `courier_pay.coefficient_used` | `numeric(3,2)` | значения 0.10–0.50 (BR-6.4) |
| `*.rating` | `numeric(3,2)` | 0–5 с двумя знаками |
| `client.password` | `varchar(32)` | MD5-хэш: 32 символа (BR-2.3) |
| `*.phone_number` | `char(12)` | `+7` и 10 цифр |
| `restaurant.inn` / `ogrn` / `settlement_account` | `char(10)` / `char(13)` / `char(20)` | ИНН юрлица, ОГРН, расчётный счёт: фиксированной длины |

---

## Нормализация (3НФ)

```
Все 12 таблиц находятся в 3НФ.

1НФ. Значения атомарны. Два места из erd.md были составными, и они разложены на отдельные столбцы:
  - courier: ФИО -> last_name, first_name, patronymic (BR-6.1);
  - restaurant: реквизиты -> inn, ogrn, settlement_account (BR-4.5).
  Реквизиты оставлены столбцами таблицы restaurant, а не вынесены в отдельную таблицу: у ресторана один набор
  реквизитов, и BR-4.5 прямо говорит, что их историю не ведём.

2НФ. У таблиц с составным ключом (client_address, courier_pay, order_structure) неключевые атрибуты зависят от всего ключа:
  quantity — от (order_id, product_id); amount и coefficient_used — от (year, month, courier_id).
  Поэтому restaurant_id не дублируется в order_structure: он зависел бы только от order_id (см. ловушку 3).

3НФ. Транзитивных зависимостей нет: данные клиента, ресторана, курьера и адреса не повторяются в заказе, только ссылки.

Особые случаи:
- order.commission = 10% от order_amount — вычисляемый столбец (GENERATED ... STORED): значение хранится, но разойтись с источником не может.
- order.total_amount = order_amount + commission — вычисляемый столбец по той же причине.
- order.order_amount — стоимость блюд на момент заказа. Её можно вычислить по order_structure и product.price, но цена блюда
  меняется, а позиция заказа цену не хранит, поэтому сумма фиксируется в заказе как исторический факт.
- product.rating и restaurant.rating — агрегаты по review. Это осознанная денормализация для скорости чтения;
  поддерживаются триггером rating_change, без отзывов значение NULL.
- order.pay_status — агрегат по попыткам оплаты (BR-8.7), поддерживается триггером payment_sync_pay_status.
- review.entity_id + entity_type — полиморфизм. 3НФ не нарушает; корректность обеспечивается триггером (см. ловушку 4).
```

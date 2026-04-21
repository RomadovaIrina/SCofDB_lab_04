# Отчёт по лабораторной работе №4
## Идемпотентность платежных запросов в FastAPI

**Студент:** _Ромадова Ирина Олеговна_  
**Группа:** _БПМ-22-ПО-3_  
**Дата:** _10.04.2026_

## 1. Постановка сценария
_TODO: Опишите сценарий \"запрос на оплату -> обрыв сети -> повторный запрос\"._
1. Клиент отправляет запрос на оплату
2. начинается обработка запроса
3. сеть обрывается на клиенте или например timeout
4. но запрос на сервере обработан
5. клиент не понимает прошла ли оплата, тк не получает ответ от сервера
6. отправляет запрос еще раз

7. в этот момент возможно несколько вариантов:
   - Повторная оплата -- недопустимо
   - отправка ошибки 

1. Что происходит без защиты: сервер не понимает, что запрос не новый, а повторный и происходит независимая обработка
2. Почему возможна повторная обработка одного и того же намерения клиента: отсутствует возможность идентификации того что запросы одинакоые на самом деле, проблемы на уровне сети не дают гарантий успешноой отправки ответа

## 2. Реализация таблицы idempotency_keys
```sql
CREATE TABLE idempotency_keys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    idempotency_key VARCHAR(255) NOT NULL,
    request_method VARCHAR(16) NOT NULL,
    request_path TEXT NOT NULL,
    request_hash TEXT NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'processing',
    status_code INTEGER,
    response_body JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,
    CONSTRAINT idempotency_status_check 
    CHECK (status IN ('processing', 'completed', 'failed'))


AlTER TABLE idempotency_keys
ADD CONSTRAINT unique_idempotency_key_per_endpoint
UNIQUE (idempotency_key, request_method, request_path);



);
```
описание:
- 1idempotency_key` - сам ключ для реализации идемпотентности для передачи в хэдерах запроса, для связи повторных запросов с уже обработанными
- `request_method` и `request_path` - в связке служать для индентификации запроса. В данной лабораторной конкретно для определения method == POST path==retry demo
- `request_hash` -  SHA256 это хэш тела запроса для установчления факта совпадения повторного запроса с уже выполненным. То  есть если ключ такой эе но хэш отличается, значит клиент пытался использовать тот же ключ для другого payload.
- `status` - статус запроса=
  1. processing - запрос обрабатывается
  2. success - обработка завершена успешно
  3. failef - ошибка
- `expires_at` - время жизни ключа, это нужно чтобы не хранить старые ненужные ключи и не увеличивать число записей в
- `status_code` - HTTP код ответа запроса

ограничения:
- `unique_idempotency_key_per_endpoint`: гарантиця уникальност ключа в рамках endpoint — защита от дубликатов на вставке.
- `idx_key_lookup`: индекс для быстрого поиска по ключу, методу и пути
- `idx_key_expires_at`: индекс для эффективной очистки недействительных записей

Укажите:
- какие поля используются для идентификации запроса: `idempotency_key` + `request_method` + `request_path`
- как хранится кэш ответа: `status_code` + 
- какие ограничения/индексы добавлены и зачем.

## 3. Реализация middleware
_TODO: Опишите алгоритм middleware по шагам._

Минимум:
1. ЧЧтение Idempotency-Key.
   - проверка что метод это POST и путь /api/payments (как раз это request_method + request_path)
   - - из хэдеров читаем Idempotency-Key.
2. Проверка существующей записи.
   - определение хэша запроса через `self.build_request_hash(result)` чтобы определить совпал ли текущий с тем что уже был обработан и сохранен
3. Обработка кейса \"тот же key + другой payload\".
   - если в таблице ключей нашлась запись по комбинации `idempotency_key` + `request_method` + `request_path`, то это говорит о том что ключ пытаются использовать повторно для другого запроса
   - такого быть не должно, потому что один ключ - один логический запрос => ошибка 409 Conflict 
4. Сохранение результата первого запроса.
   - если записи по этому ключу не нашлось, то воздается новая запись, статус которой будет processing
   - Потом запрос передается дальше в обработку
   - После завершения работы endpoint-а, из ответа ивзлекаем `status_code` и `responce_body` и апдейт таблицы с установкой статуса completed
6. Возврат кэшированного ответа при повторе.
   - если при повторном запросе имеется запись с таким же ключом, путем и хэшом, то это говорит о то м что хапрос обрабатывается повторно
   - здесь нет обращения к  endpoint-у повторно, а просто возврат ответа из таблицы тк он уже сохранен

## 4. Демонстрация без защиты
_TODO: Приведите результаты сценария без идемпотентности._
запуск:
```bash
docker compose exec -T backend pytest app/tests/test_retry_without_idempotency.py -v -s
```

Вывод
```
{'id': 'b5430965-cdc3-4326-ac5d-e53d634d3cca', 'order_id': '9d305c0a-0663-48c4-af30-1d5034ef6ef5', 'status': 'paid', 'changed_at': datetime.datetime(2026, 4, 15, 19, 18, 54, 794866)}
{'id': 'aea9d090-18e3-4efe-a307-bf3f1a18e28c', 'order_id': '9d305c0a-0663-48c4-af30-1d5034ef6ef5', 'status': 'paid', 'changed_at': datetime.datetime(2026, 4, 15, 19, 18, 54, 813179)}
PASSED

```
видим, что у одного ордер айди в разные таймстэмпы стоит статус paid

## 5. Демонстрация с Idempotency-Key
_TODO: Приведите результаты сценария с одним и тем же ключом._

Укажите:
- первый ответ;
- повторный ответ;
- признак того, что повторный ответ кэшированный;
- подтверждение, что повторного списания не было.


Запрос 1:
```json
response_1: {
   'status': 'paid',
   'success': True,
   'message': 'Retry demo payment succeeded (unsafe)',
   'order_id': '49db90c3-39ca-449d-8e97-4b6526c06e88',
}
```

Запрос 2:
```json
 response_2: {
   'status': 'paid',
   'success': True,
   'message': 'Retry demo payment succeeded (unsafe)',
   'order_id': '49db90c3-39ca-449d-8e97-4b6526c06e88',
}
```


Логи:
```
026-04-10 10:04:20,256 INFO sqlalchemy.engine.Engine [generated in 0.00012s] (200, '{"success": true, "message": "Retry demo payment succeeded (unsafe)", "order_id": "49db90c3-39ca-449d-8e97-4b6526c06e88", "status": "paid"}', 'fixed-key-123', 'POST', '/api/payments/retry-demo')
2026-04-10 10:04:20,257 INFO sqlalchemy.engine.Engine COMMIT
2026-04-10 10:04:20,258 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2026-04-10 10:04:20,259 INFO sqlalchemy.engine.Engine SELECT id, status, response_body, request_hash, status_code
                    FROM idempotency_keys
                    WHERE idempotency_key = $1 
                    AND request_method =  $2
                    AND request_path = $3
                    FOR UPDATE
                
2026-04-10 10:04:20,259 INFO sqlalchemy.engine.Engine [cached since 0.1307s ago] ('fixed-key-123', 'POST', '/api/payments/retry-demo')

```
видим, что еще вернулся fixed-key-123


## 6. Негативный сценарий
_TODO: Один и тот же ключ с разным payload._


Запрос 1:
```json
 RESPONSE 1: {
'success': True,
'message': 'Retry demo payment succeeded (unsafe)',
'order_id': '3f11263f-091b-434a-b347-d8d1962f3c73',
'status': 'paid'}
```

Запрос 2: --> как раз ошибка 409
```json
 RESPONSE 2: {
'error': 'Conflict: usage of the same key with different payload'
}
```

Логи
```

2026-04-10 10:04:20,425 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2026-04-10 10:04:20,425 INFO sqlalchemy.engine.Engine SELECT id, status, response_body, request_hash, status_code
                    FROM idempotency_keys
                    WHERE idempotency_key = $1 
                    AND request_method =  $2
                    AND request_path = $3
                    FOR UPDATE
                
2026-04-10 10:04:20,425 INFO sqlalchemy.engine.Engine [cached since 0.2969s ago] ('fixed-key-123', 'POST', '/api/payments/retry-demo')
2026-04-10 10:04:20,426 INFO sqlalchemy.engine.Engine COMMIT

 RESPONSE 2: {'error': 'Conflict: usage of the same key with different payload'}

```


Ожидаемо:
- ошибка конфликта (`409 Conflict` или эквивалент).

## 7. Сравнение с решением из ЛР2 (FOR UPDATE)
_TODO: Сравните подходы по сути и по UX._

| Характеристика | FOR UPDATE (ЛР2) | Idempotency-Key (ЛР4) |
|----------------|------------------|------------------------|
| Цель механизма | Защита от race condition  | Защита от повторных запросов из-за проблем сети  |
| Повтор запроса | Второй запрос видит, что заказ уже оплачен (success=False) | Второй запрос получает кэшированный ответ первого запроса (success=True) |
| HTTP статус | 200, но неуспешно | 200 и X-Idempotency-Replayed |
| Уровень зашиты | Уровень БД | Уровень API -- кэширование ответов|

Как когда использовать: 
- Подходы решают разные задачи на разных уровнях => нет конфликтов именно между ними. Использование защиты на уровне Бд с FOR UPDATE + защита от сетевых ошибок с помощью идемпотентных ключей обеспечит более высокую наджежность 


## 8. Выводы
1. использование Middleware позволяет создать дополнительный уровень защиты решает проблему повторных запросов, которые могут возникнуть из-за сетевых проблем или тайм-аутов.
2. Тк генерацией идемпотентного ключа занимается клиент, то ответственность за его уникальность тоже. и также клиент отправляет запрос с уникальным ключом, сервер может кэшировать его ответ для последующей обработки.
3. Использование вместе с for update позволяет добить защиты от повторных запросов на уровне клиента и на уровне бд и так добиться лушчшей надежности

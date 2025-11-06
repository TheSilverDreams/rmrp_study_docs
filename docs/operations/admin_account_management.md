 # Управление аккаунтами администраторов (Superadmin API)

> Все конечные точки из этого раздела требуют аутентификации и права `superadmin`.  
> В запросах используйте заголовок `Authorization: Bearer <JWT>`.

## Доступные права для сотрудников

Метаданные с перечнем доступных разрешений и их матрицей можно получить так:

```http
GET /api/superadmin/administrators/metadata
```

Ответ:

```json
{
  "permissions": ["students.read", "students.create", "students.reset", "students.delete", "students.group", "staff.manage"],
  "defaults": {
    "admin": ["students.read", "students.create", "students.reset"],
    "superadmin": ["students.read", "students.create", "students.reset", "students.delete", "students.group", "staff.manage"]
  },
  "matrix": {
    "students.group": ["admin", "superadmin"],
    "staff.manage": ["superadmin"]
  }
}
```

Новое право `students.group` позволяет назначать учебные группы ученикам из карточки пользователя.

## Список администраторов

```http
GET /api/superadmin/administrators
```

Ответ:

```json
{
  "users": [
    {
      "id": "uuid",
      "login": "admin",
      "role": "admin",
      "displayName": "Администратор",
      "staticId": null,
      "gameId": null,
      "server": null,
      "createdAt": "2025-10-10T09:00:00Z",
      "courseGroup": null,
      "tags": [],
      "permissions": ["students.read", "students.create", "students.reset"]
    }
  ]
}
```

## Создание администратора

```http
POST /api/superadmin/administrators
Content-Type: application/json

{
  "login": "admin.vasya",
  "displayName": "Василий Петров",
  "role": "admin",
  "password": "при необходимости можно задать свой пароль",
  "permissions": ["students.read", "students.group"]
}
```

Примечания:

- `role`: `admin` или `superadmin`.
- Пароль можно не указывать — тогда API сгенерирует его автоматически.
- `permissions` опциональны. Если поле не передано, используются значения по умолчанию для роли.

Ответ (`201 Created`):

```json
{
  "user": {
    "id": "uuid",
    "login": "admin.vasya",
    "role": "admin",
    "displayName": "Василий Петров",
    "permissions": ["students.read", "students.group"]
  },
  "password": "СгенерированныйПароль123",
  "passwordGenerated": true
}
```

> Пароль показывается один раз. Сохраните его до закрытия карточки.

## Просмотр / обновление администратора

Получение карточки:

```http
GET /api/superadmin/administrators/{userId}
```

Обновление имени, роли или набора прав:

```http
PATCH /api/superadmin/administrators/{userId}
Content-Type: application/json

{
  "displayName": "Alex Doe",
  "role": "superadmin",
  "permissions": ["students.read", "students.create", "students.reset", "students.delete", "students.group", "staff.manage"]
}
```

> Нужно передать хотя бы одно поле `displayName`, `role` или `permissions`.  
> Проверяются ограничения: нельзя оставить систему без супер-администратора и нельзя назначить права, не соответствующие роли.

## Сброс пароля администратора

```http
POST /api/superadmin/administrators/{userId}/reset-password
Content-Type: application/json

{
  "password": "NewSecurePassword123" // опционально
}
```

- Если пароль не передан, генерируется автоматически.
- Ответ содержит логин, новый пароль и признак, был ли он сгенерирован.

## Удаление администратора

```http
DELETE /api/superadmin/administrators/{userId}
```

Ограничения:

- Нельзя удалить самих себя (`400`).
- Должен остаться хотя бы один супер-администратор (`400` при последнем).
- Удаление учеников через этот endpoint запрещено (`400`).
- Успешный ответ — `204 No Content`.

## Журнал авторизаций персонала

Полный список:

```http
GET /api/superadmin/logins?limit=50&cursor=...
```

Логи конкретного пользователя:

```http
GET /api/superadmin/users/{userId}/logins?limit=20&cursor=...
```

Параметры:

- `limit` — количество записей (по умолчанию 50, максимум 200).
- `cursor` — отметка времени (`createdAt`) последней записи из предыдущей выдачи.

Ответ:

```json
{
  "logs": [
    {
      "id": "uuid",
      "userId": "uuid",
      "login": "admin",
      "displayName": "Администратор",
      "role": "admin",
      "ipAddress": "10.10.10.10",
      "userAgent": "Mozilla/5.0 ...",
      "createdAt": "2025-10-10T09:10:00Z"
    }
  ],
  "nextCursor": "2025-10-10T08:55:00Z"
}
```

> Логи хранятся 30 дней. Возвращается `nextCursor`, который нужно передать в следующем запросе для получения следующей порции данных.

## Важные замечания

- Все критичные маршруты требуют JWT и права `superadmin`. Дополнительно:
  - `/logins` и `/users/{id}/logins` — право `staff.manage`.
- Новый `students.group` даёт возможность админам изменять учебную группу в интерфейсе (страница карточки ученика).
- После изменения прав обязательно обновляйте токен в приложении — фронтенд подтянет новые разрешения только после повторной авторизации.

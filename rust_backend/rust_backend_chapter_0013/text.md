# Глава 1.3. Роутинг

Axum использует декларативный подход к маршрутизации. Вы не пишете `if path == "/users"`, а описываете таблицу соответствий `путь + метод -> функция`. `Router` - обычный тип, поэтому маршруты можно собирать из нескольких функций или модулей.
## Маршруты
Базовый синтаксис выглядит так:
```rust
use axum::{routing::get, Router};

let app = Router::new()
    .route("/", get(root))
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(get_user).delete(delete_user));
```

Функция `route` принимает путь и структуру `MethodRouter` (таблицу соответствий между HTTP-методами и handlers). Например, `.route("/users", get(list_users).post(create_user))` означает
```
GET  /users -> list_users
POST /users -> create_user
```

Модуль `axum::routing` предоставляет функции для каждого HTTP-метода: `get`, `post`, `put`, `patch`, `delete` и т.д. Отдельно объявлять `head` не нужно, `get` обслуживает и его. Для любого метода есть `any`, для нескольких сразу - `on`. todo

Один путь нельзя добавить в роутер дважды:
```rust
Router::new()
    .route("/users", get(list_users))
    .route("/users", post(create_user)); // panic
```
## Параметры пути

Часть пути можно сделать параметром, указав его имя в фигурных скобках. Значение извлекается экстрактором `Path`:
```rust
use axum::extract::Path;

async fn get_user(Path(id): Path<u64>) -> String {
    format!("user {id}")
}

let app = Router::new().route("/users/{id}", get(get_user));
```

Если параметров несколько, их извлекают кортежем в том порядке, в котором они идут в пути:
```rust
// маршрут /users/{user_id}/posts/{post_id}
async fn get_post(Path((user_id, post_id)): Path<(u64, u64)>) {}
```

Захватить весь оставшийся путь с помощью wildcard:
```rust
.route("/assets/{*path}", get(serve_asset))
```
Например, `/assets/css/main.css` передаст в `path` значение `css/main.css`.

Подробнее про экстракторы в главе 1.5.
### Приоритет маршрутов

```rust
.route("/users/me", get(current_user))
.route("/users/{id}", get(get_user))
```

Запрос `GET /users/me` попадёт в `current_user`. Причём порядок объявления маршрутов не важен, поэтому `/users/{id}` и `/users/me` можно поменять местами без изменения поведения. Также маршруты, совпадающие по структуре, запрещены, даже если у параметров разные имена: `/users/{id}` и `/users/{user_id}` в одном роутере вызовут панику при запуске.
## 404, 405 и fallback

Axum различает два случая:
- `404 Not Found` - такого пути нет
- `405 Method Not Allowed` - путь существует, но для него не зарегистрирован данный HTTP-метод

Например, если зарегистрирован только `GET /users`, то `POST /users` получит `405` и заголовок `Allow` со списком доступных методов, а `GET /unknown` получит `404`. Полный список HTTP-методов и кодов ответа можно посмотреть на [MDN](https://developer.mozilla.org/ru/docs/Web/HTTP).

Учтите, что `/users` и `/users/` - это разные пути и автоматического перенаправления между ними axum не делает. Для API часто полезно настроить собственный `fallback` - хендлер, который вызывается, когда ни один маршрут не подошёл:
```rust
use axum::{http::StatusCode, response::IntoResponse, Json};
use serde_json::json;

async fn not_found() -> impl IntoResponse {
    (StatusCode::NOT_FOUND, Json(json!({ "error": "not found" })))
}

let app = Router::new()
    .route("/", get(root))
    .fallback(not_found);
```
## Основные методы `Router`

Помимо `route`, у `Router` есть несколько методов, которые часто используются при построении приложения.

1) `merge` - объединяет два роутера, сохраняя их пути:
```rust
let users = Router::new().route("/users", get(list_users));
let posts = Router::new().route("/posts", get(list_posts));

let app = Router::new().merge(users).merge(posts); 
```

Если в объединяемых роутерах окажутся одинаковые пути, будет паника при запуске.

2) `nest` - монтирует роутер под определённым префиксом:
```rust
let users = Router::new()
    .route("/", get(list_users))
    .route("/{id}", get(get_user));

let app = Router::new()
    .nest("/users", users);
```

В результате маршруты будут:
```
GET /users
GET /users/{id}
```

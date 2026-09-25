# Глава 1.2. Сериализация и десериализация JSON

Практически каждая ручка в API это разбор json на входе и сборка его на выходе. В экосистеме Rust стандартом для cериализация и десериализация является `serde`.

Сериализация - преобразование данных из Rust во внешний формат (`Rust -> JSON`), десериализация - обратный процесс (`JSON -> Rust`).

Сам `serde` не умеет работать ни с JSON, ни с другими форматами. Он задаёт общий интерфейс, а конкретные форматы реализуются отдельными крейтами: `serde_json`, `toml`, `rmp-serde` и другими. Благодаря этому одну структуру можно использовать с разными форматами.

Добавьте новые зависимости в `Cargo.toml`:
```bash
cargo add serde --features derive 
cargo add serde_json
```
## Основные трейты

У serde два трейта: `Serialize` для сериализации и `Deserialize` для десериализации. Вручную их обычно не реализуют, а используют макрос `derive`:
```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)] // генерирует реализацию трейтов на этапе компиляции
struct User {
    id: i64,
    name: String,
}

let json = serde_json::to_string(&user)?;      // превращает rust-значение в json
let user: User = serde_json::from_str(&json)?; // json в rust-значение
```
## JSON в Axum

В хендлерах эти функции вызывать не нужно. В Axum есть экстрактор `Json<T>`: в аргументах хендлера она разбирает тело запроса, а в возвращаемом значении сериализует ответ и выставляет `content-type: application/json`.
```rust
use axum::Json;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    name: String,
}

#[derive(Serialize)]
struct UserResponse {
    id: i64,
    name: String,
}

async fn create_user(Json(user): Json<CreateUser>) -> Json<UserResponse> {
    Json(UserResponse { id: 1, name: user.name })
}
```

Здесь Axum автоматически десериализует тело запроса в `CreateUser`, а возвращаемый `Json<UserResponse>` сериализует обратно в json. 

Если тело не удалось разобрать, хендлер не вызывается, Axum сам вернёт ошибку:
- `415` - нет заголовка `content-type: application/json`
- `400` - синтаксическая ошибка в JSON
- `422` - JSON корректный, но поля не те или не того типа

В структурах запроса используйте `String`, а не `&str`, так как `Json<T>` требует тип, который ничего не заимствует из тела запроса (срезы не принимаются).
## Основные атрибуты

По умолчанию serde работает по двум правилам: все поля обязательные (если поля нет, будет ошибка), а лишние поля молча игнорируются. Большинство атрибутов нужны, чтобы изменить это поведение.

1) Переименование. `rename_all` - переименовывает все поля в нужный формат, а `rename` - одно:
```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct User {
    first_name: String,
    #[serde(rename = "type")]
    kind: String,
}
// Теперь в json все поля буду в camelCase: {"firstName":"...","type":"..."}
```
Кроме `camelCase` есть `PascalCase`, `kebab-case`, `SCREAMING_SNAKE_CASE` и другие.

2) Необязательные поля
	- `Option<T>` - отсутствующее поле и `null` одинаково дают `None`
	- `default` - подставляет значение из трейта `Default` или из своей функции
	- `skip_serializing_if` - не выводит поле при выполнении условия, без него `None` станет `"email": null`
```rust
#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    #[serde(default)]
    is_admin: bool, // false
    #[serde(default = "default_role")]
    role: String,   // "user"
    #[serde(skip_serializing_if = "Option::is_none")]
    email: Option<String>,
}

fn default_role() -> String {
    "user".to_string()
}
// на входе достаточно {"name":"Alice"}
// на выходе {"name":"Alice","is_admin":false,"role":"user"}
```

3) Пропуск полей. `skip` полностью исключает поле из сериализации и десериализации. При десериализации вместо значения используется `Default::default()`. Это удобно для внутренних полей, которые не должны передаваться через API:
```rust
#[derive(Serialize, Deserialize)]
struct User {
    name: String,
    #[serde(skip)]
    password_hash: String,
}
```

4) Строгая проверка. Из-за того, что лишние поля игнорируются, опечатка клиента проходит незамеченной. `deny_unknown_fields` превращает неизвестное поле в ошибку:
```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
struct UpdateUser {
    name: Option<String>,
}
// {"nmae":"Bob"}
// без атрибута: name = None, обновление ничего не сделало, ответ 200 OK
// с атрибутом: ошибка unknown field `nmae`
```

5) Разворачивание вложенных полей. `flatten` помещает поля вложенной структуры на один уровень с остальными:
```rust
#[derive(Serialize, Deserialize)]
struct User {
    name: String,

    #[serde(flatten)]
    metadata: Metadata,
}

#[derive(Serialize, Deserialize)]
struct Metadata {
    created_at: String,
}

// {"name":"Alice","created_at":"2026-09-24"}
```

6) Перечисления. Перечисление без данных сериализуется в строку, это полезно например для статусов. У перечисления с данными по умолчанию получается `{"Click":{"x":10}}`, а в API обычно ждут отдельное поле с типом, это задаётся атрибутом `tag`:
```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
enum OrderStatus {
    New,        // new
    InProgress, // "in_progress"
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
enum Event {
    Click { x: i32 }, // {"type":"click","x":10}
}
```
## `serde_json::Value` и `json!`

Когда структура JSON заранее неизвестна, используют `serde_json::Value` - тип, представляющий произвольное JSON-значение. 
```rust
let value: Value = serde_json::from_str(r#"{"name": "Alice"}"#)?;

println!("{}", value["name"]);   // "Alice" - вместе с кавычками
value["name"].as_str();          // Some("Alice")
```

При индексировании несуществующего ключа `Value` вернёт `null`:
```rust
println!("{}", value["age"]); // null
```

Макрос `json!` позволяет создавать `Value` из JSON-подобного литерала. Он удобен для небольших ответов и тестов:
```rust
async fn health() -> Json<Value> {
    Json(json!({ "status": "ok" }))
}
```

Злоупотреблять `Value` не стоит. Если структура JSON известна заранее, лучше описать её отдельным типом. Так вы получаете проверку структуры на этапе компиляции и удобное автодополнение, а не ошибки доступа к полям во время выполнения.

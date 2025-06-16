# **Неделя 1-2: Rust Essentials**

**Цель:**  
Научиться писать безопасный Rust-код без борьбы с компилятором. Понимать `ownership`, `borrowing`, `lifetimes` на уровне интуиции.

---

## **Теоретические материалы:**

1. **Библия Rust (обязательно):**

   - [The Rust Book: Глава 4 (Ownership)](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
   - [The Rust Book: Глава 10.3 (Lifetimes)](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)  
     _Конспект ключевых концепций:_

   ```markdown
   - Ownership Rules:
     1. У каждого значения есть owner
     2. Только один owner одновременно
     3. Owner выходит из области видимости → значение дропается
   - Borrowing:
     - &T → shared reference (много читателей)
     - &mut T → mutable reference (один писатель)
   - Lifetimes: аннотации (`'a`) гарантируют, что ссылки валидны дольше, чем используются
   ```

2. **Глубокое погружение (видео):**

   - [Jon Gjengset: "Crust of Rust #3 - Ownership, Borrowing, Lifetimes"](https://www.youtube.com/watch?v=8M0QfLUDaaA)  
     _Разбор с визуализацией памяти и ассемблером._

3. **Практические паттерны (статьи):**
   - [Rust Patterns: Minimizing Clone()](https://www.lpalmieri.com/posts/2024-02-01-rust-patterns-minimize-cloning/)
   - [Lifetime Annotations Cheatsheet](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md)

---

## **Практические задания:**

**Задание 1: CLI-парсер логов**  
_Цель:_ Применить ownership/borrowing для обработки данных без лишних аллокаций.

```rust
// Пример входных данных (access.log)
// 192.168.1.1 - - [15/Jun/2025:10:30:45 +0000] "GET /index.html HTTP/1.1" 200 1234
struct LogEntry<'a> {
    ip: &'a str,    // Используем срезы вместо String!
    status: u16,
    path: &'a str,
}

impl<'a> LogEntry<'a> {
    fn parse(line: &'a str) -> Result<Self, &'static str> { /* ... */ }
}

fn main() -> Result<(), Box<dyn Error>> {
    let file = File::open("access.log")?;
    let reader = BufReader::new(file);

    let mut stats: HashMap<&str, u32> = HashMap::new(); // Учимся работать с lifetime в коллекциях!

    for line in reader.lines() {
        let entry = LogEntry::parse(&line?)?;
        *stats.entry(entry.path).or_insert(0) += 1;
    }

    Ok(())
}
```

**Требования:**

- Использовать срезы (`&str`) вместо `String` где возможно
- Избегать `.clone()` (максимум 3 раза в коде)
- Обработать ошибки парсинга через `Result`
- Вывести ТОП-5 самых частых путей

---

**Задание 2: Кеш с проверкой lifetimes**  
_Цель:_ Закрепить работу с аннотациями времени жизни.

```rust
struct Cache<'a, T> {
    data: HashMap<String, &'a T>, // Храним ссылки!
}

impl<'a, T> Cache<'a, T> {
    fn new() -> Self { /* ... */ }

    // Метод должен гарантировать, что значение живёт дольше кеша
    fn insert(&mut self, key: String, value: &'a T) { /* ... */ }

    // Вернуть ссылку без нарушения безопасности
    fn get(&self, key: &str) -> Option<&&'a T> { /* ... */ }
}

// Тест, который должен скомпилироваться
#[test]
fn test_no_dangling() {
    let mut cache = Cache::new();
    let value = String::from("hello"); // Владеемое значение

    {
        cache.insert("key1".to_string(), &value);
        let cached = cache.get("key1").unwrap();
        assert_eq!(*cached, "hello");
    } // `value` дропается здесь → компилятор ДОЛЖЕН отловить ошибку!
}
```

**Требования:**

- Код компилируется без `unsafe`
- Тест `test_no_dangling` **не должен** компилироваться (доказательство проверки lifetimes)
- Написать тест, где кеш переживает значения

---

## **Дополнительные ресурсы:**

1. **Интерактивные тренажёры:**

   - [Rustlings: Exercises 4-15 (ownership, lifetimes)](https://github.com/rust-lang/rustlings)
   - [Exercism: "High-Scores" (практика с коллекциями)](https://exercism.org/tracks/rust/exercises/high-scores)

2. **Визуализация памяти:**

   - [Ownership Memory Model Sandbox](https://rust-book.cs.brown.edu/ch04-01-what-is-ownership.html#ownership-and-functions)

3. **Чеклист для самопроверки:**

   ```markdown
   [ ] Понимаю разницу между String и &str
   [ ] Могу объяснить, почему этот код не компилируется:
   let s = String::from("hello");
   let ref1 = &s;
   let ref2 = &s;
   let mut ref3 = &mut s; // ← здесь ошибка!
   [ ] Знаю, когда использовать .to_owned() vs .clone()
   [ ] Могу добавить lifetime-аннотации в структуру с ссылками
   ```

---

## **Критерии оценки:**

1. **Парсер логов:**

   - Нет паники при невалидных данных
   - Корректная работа со срезами (0 копирований строк)
   - Время обработки 100 МБ лога < 2 сек (benchmark)

2. **Кеш:**
   - Компилятор ловит dangling references
   - Документация с примерами безопасного использования
   - Тест на попытку вставить значение с коротким lifetime

---

**Совет ментора:**

> "Не боритесь с компилятором — учитесь у него. Каждая ошибка borrow checker’а раскрывает нюанс системы владения. Используйте `cargo clippy -- -W clippy::pedantic` как строгого наставника."

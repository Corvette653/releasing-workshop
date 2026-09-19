# Wstęp

Zacznijmy od krótkiego technicznego wprowadzenia — zrozummy, czym jest aplikacja full stack i jak działa.

## Z czego składa się aplikacja full stack

<img src="full-stack-restaurant.jpeg" width="600">

### Frontend
Najbardziej oczywista część. To, co użytkownik widzi i z czym wchodzi w interakcję — nasza sala dla gości.
<img src="app-screenshot.png" width="500">

### Backend
Kuchnia naszej aplikacji. Odbiera żądania, sprawdza dane i reguły, wykonuje obliczenia i odpowiednio odpowiada. Rozmawia też z bazą danych, żeby zapisywać i odczytywać informacje (czego frontend nie potrafi zrobić sam).

### Baza danych
Przechowuje wszystkie informacje, których potrzebuje nasza aplikacja.

<img src="db-content.png" width="500">

## Porty w komputerze

<img src="mailbox-wall.png" width="600px">

Nasz komputer jest jak blok mieszkalny. Każde mieszkanie ma skrzynkę pocztową z numerem (portem). Kiedy wysyłamy list, nie wystarczy podać adresu budynku — trzeba też podać numer mieszkania.

## Jak to wszystko działa razem?

<img src="request-flow.png" width="700">

Kiedy otwierasz plik HTML w przeglądarce, ta go wczytuje i interpretuje jego kod. W środku znajdzie strukturę strony oraz odnośniki do CSS (stylów) i JS (skryptów), więc wczyta i zinterpretuje również je.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Todo App</title>
  <link rel="stylesheet" href="style.css"><!-- wczytujemy style -->
</head>
<body>
  <h1>Todo List</h1>

  <form id="todo-form">
    <input type="text" id="todo-title" placeholder="What needs doing?" required>
    <select id="todo-priority">
      <option value="1">low</option>
      <option value="2" selected>medium</option>
      <option value="3">high</option>
    </select>
    <button type="submit">Add</button>
  </form>

  <ul id="todo-list"></ul>

  <script src="config.js"></script><!-- wczytujemy konfigurację -->
  <script src="app.js"></script><!-- wczytujemy skrypty -->
</body>
</html>
```

<img src="loading-front.png" width="600px">
To sprawi, że strona załaduje się użytkownikowi. Wewnątrz skryptu `app.js` przeglądarka natrafi na wywołanie funkcji `loadTodos()` — to nasz klient składający zamówienie. Teraz kelner musi pójść do kuchni i przekazać je szefowi.

```js
API_URL = "http://localhost:8000";

async function loadTodos() {
  const res = await fetch(`${API_URL}/todos`);
  const todos = await res.json();
  ...
}

loadTodos();
```

Zanim przejdziemy do backendu, zrozummy trochę lepiej, jak frontend i backend ze sobą rozmawiają.

`fetch()` to wbudowana funkcja JS, która pozwala wysyłać żądania HTTP na podany adres. W naszym przypadku wysyłamy domyślne żądanie `GET` (zwykle oznaczające "pobierz coś") na adres `localhost:8000/todos`. Trafia ono z naszej przeglądarki do karty sieciowej naszego komputera, która widzi `localhost` (tę samą maszynę) i przekazuje je na port `8000` naszego własnego komputera. Musi tam nasłuchiwać jakiś serwer, który odbierze ruch i wywoła nasz kod backendu z danymi z żądania.

```python
DATABASE_URL = "postgresql://postgres:postgres@localhost:5432/todos"
engine = create_engine(DATABASE_URL)

@app.get("/todos", response_model=list[Todo])
def list_todos():
    with Session(engine) as session:
        return session.exec(select(Todo)).all()
```

Kiedy nasz szef kuchni zostanie obudzony karteczką z tym, co ma przygotować, idzie do lodówki (bazy danych). Najważniejszą częścią tej operacji jest zrozumienie connection stringa (`DATABASE_URL`)

- `postgresql` — nazwa protokołu / typu połączenia; mówi, że będziemy używać PostgreSQL
- `postgres` przed `:` — nazwa użytkownika bazy danych
- `postgres` po `:` — hasło użytkownika
- `localhost` — adres serwera bazy danych; tutaj baza działa lokalnie na naszym komputerze
- `5432` — port, przez który łączymy się z bazą danych
- `todos` — nazwa konkretnej bazy danych, z którą chcemy pracować

Backend korzysta z biblioteki, żeby rozmawiać z bazą danych, i tym razem zamiast HTTP używany jest własny protokół PostgreSQL — ale ta część, w której żądanie trafia do karty sieciowej, a potem wraca do naszego komputera (`localhost`) i na port `5432`, wygląda tak samo. Stamtąd do gry wchodzi baza danych. Sprawdza, czy kombinacja użytkownik-hasło się zgadza, wykonuje podane zapytanie i tą samą drogą, tylko w drugą stronę, odsyła wynik do naszego backendu.

W tym przypadku backend po prostu przekazuje dalej do frontendu to, co dostał z bazy danych. Niewiele tu gotowania.

```js
async function loadTodos() {
  const res = await fetch(`${API_URL}/todos`);
  const todos = await res.json();

  list.innerHTML = "";
  for (const todo of todos) {
    const li = document.createElement("li");

    const checkbox = document.createElement("input");
    checkbox.type = "checkbox";
    checkbox.checked = todo.done;
    checkbox.addEventListener("change", () => toggleTodo(todo.id));

    const span = document.createElement("span");
    span.textContent = todo.title;
    if (todo.done) span.classList.add("done");

    const deleteBtn = document.createElement("button");
    deleteBtn.textContent = "Delete";
    deleteBtn.addEventListener("click", () => deleteTodo(todo.id));

    li.append(checkbox, span, deleteBtn);
    list.append(li);
  }
}
```

Frontend dostaje dania i dla każdego z nich przygotowuje element listy z checkboxem (talerz), tytułem (widelec) i przyciskiem usuwania (nóż). Potem składa to wszystko razem i pokazuje użytkownikowi. Teraz użytkownik może wejść w interakcję z aplikacją i złożyć nowe zamówienia (dodać nowe zadania) albo zmienić istniejące (oznaczyć jako zrobione albo usunąć). Każda z tych akcji wywoła nowe żądanie do backendu, które pójdzie do bazy danych i z powrotem, dokładnie tak jak poprzednio.

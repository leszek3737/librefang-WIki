# Root — rustfmt.toml

# Root — `rustfmt.toml`

Konfiguracja formatowania na poziomie workspace dla projektu LibreFang w języku Rust. Ten plik nie ma wpływu na zachowanie w czasie wykonania; określa sposób, w jaki `cargo fmt` i `rustfmt` przepisują kod źródłowy we **wszystkich** crate'ach w workspace.

## Cel

Jednolite formatowanie jest egzekwowane przez CI, a ta konfiguracja jest jedynym źródłem prawdy dla tych reguł. Każdy wkład musi przejść `cargo fmt --check` przed scaleniem, a poniższe ustawienia definiują dokładnie, co oznacza „przejście" sprawdzenia.

Plik znajduje się w katalogu głównym workspace, aby miał zastosowanie automatycznie do wszystkich crate'ów członkowskich bez duplikowania w poszczególnych crate'ach.

## Ustawienia

| Ustawienie | Wartość | Efekt |
|---|---|---|
| `edition` | `2021` | Formatuje kod zakładając semantykę edycji Rust 2021 (np. zmiany w prelude, reguły przechwytywania domknięć). |
| `max_width` | `100` | Wiersze są zawijane do 100 znaków. To jest główny parametr układu wizualnego. |
| `use_field_init_shorthand` | `true` | Przepisuje `Foo { x: x }` jako `Foo { x }`. |
| `use_try_shorthand` | `true` | Przepisuje idiomy `match { Ok(v) => v, Err(e) => return Err(e) }` jako `?`. |

Wszystko, co **nie** jest wymienione tutaj, wraca do wartości domyślnych `rustfmt`.

## Jak używać

Przed wysłaniem zmian uruchom:

```sh
cargo fmt
```

To przepisuje pliki w miejscu we wszystkich członkach workspace, aby były zgodne z powyższymi regułami. Aby zweryfikować formatowanie bez modyfikowania plików — odzwierciedlając to, co sprawdza CI — uruchom:

```sh
cargo fmt --check
```

Jeśli sprawdzenie zakończy się kodem różnym od zera, CI zawiedzie na pull requeście.

## Egzekwowanie w CI

Komentarz nagłówkowy stwierdza, że jest to „Egzekwowane przez CI". Konkretnie:

- CI uruchamia `cargo fmt --check` (lub odpowiednik) jako bramkę.
- Każde odchylenie formatowania blokuje budowę, dopóki nie zostanie zastosowane `cargo fmt` i zmiany nie zostaną zatwierdzone.
- Ponieważ konfiguracja znajduje się w katalogu głównym workspace, nie ma niejednoznaczności, które reguły mają zastosowanie do którego crate'a.

## Interakcja z resztą workspace

Ten plik jest wyłącznie kontraktem formatowania. Nie:

- Wpływa na kompilację, sprawdzanie typów ani zachowanie w czasie wykonania.
- Wprowadza zależności ani flag funkcji.
- Jest konsumowany przez żadną ścieżkę kodu (graf wywołań nie ma krawędzi przychodzących, wychodzących ani wewnętrznych).

Jest jednak powiązany z toolchainem: wersja `rustfmt` w uruchomieniu CI determinuje, jak zachowują się nierozpoznane opcje lub formatowanie specyficzne dla wersji. Kontrybutorzy powinni upewnić się, że ich lokalny nightly/stable `rustfmt` odpowiada temu, czego używa CI, ponieważ wynik formatowania może się nieznacznie różnić między wersjami toolchainu.

## Kiedy zmieniać ten plik

Zaktualizuj ten plik, gdy chcesz zmienić konwencję stylu w całym workspace (np. dostosować `max_width` lub włączyć dodatkowe opcje, takie jak `imports_granularity`). Ponieważ ma zastosowanie do wszystkich crate'ów jednocześnie, takie zmiany zazwyczaj generują duży diff w całym repozytorium i powinny być skoordynowane jako dedykowany commit formatowania, a nie mieszane z pracą nad funkcjonalnością.

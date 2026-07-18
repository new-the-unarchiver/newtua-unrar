# newtua-unrar

Вынужденный форк крейта [`unrar`](https://github.com/muja/unrar.rs) — Rust-обвязка
над `libunrar` для чтения и распаковки RAR-архивов. Используется проектом
[newtua](https://github.com/new-the-unarchiver) как обычная зависимость по имени
библиотеки `unrar` (потребители кода ничего не меняют).

## Это вынужденный форк

Он существует не как самостоятельный продукт: мы его **не развиваем**, **не
принимаем в него посторонние правки** и **не даём обещаний по сопровождению**.

**Происхождение:** [muja/unrar.rs](https://github.com/muja/unrar.rs), версия 0.5.8,
лицензия `MIT OR Apache-2.0`.

**Причина форка:** в обратный вызов libunrar добавлены защиты от опасных
указателей: `UCM_PROCESSDATA` может передать нулевой буфер, а
`UCM_CHANGEVOLUMEW` может передать нулевой или невыровненный указатель на имя
тома — а если наивно скопировать по этому указателю фиксированное число
символов, то ещё и прочитать за границей буфера при переходе между томами.
Без этих проверок распаковка многотомных архивов завершает процесс через
SIGABRT.

Мы откажемся от этого форка и вернёмся на исходный крейт, как только тот начнёт
удовлетворять нашим требованиям. Если вам нужен этот код сам по себе — берите
исходный крейт, а не наш форк.

Правка отправлена в апстрим: [muja/unrar.rs#77](https://github.com/muja/unrar.rs/pull/77).
**Внимание:** в том виде, в каком запрос отправлен, он содержит только защиту
от нулевого и невыровненного указателя — защита от переполнения чтения
появилась у нас позже и в запрос ещё не внесена. Форк будет удалён, когда
апстрим примет полную версию правки и выпустит её.

## Что именно исправлено

**Целевой крейт:** `unrar` (репозиторий muja/unrar.rs), версия 0.5.8.

### Проблема

Обработка **многотомного RAR**-архива может завершить весь процесс через
`SIGABRT` на границе тома. Наблюдались два разных отказа, оба в
`Internal::callback`:

1. **Распаковка** стабильно падает с `SIGABRT`, когда данные файла пересекают
   границу тома (нулевой указатель в `UCM_PROCESSDATA`).
2. **Листинг/распаковка** *периодически* падает с `SIGABRT` при смене тома —
   из-за того, как исходный код читает широкую строку с именем следующего
   тома в `UCM_CHANGEVOLUMEW`.

### Причина

В `Internal::callback` (`src/open_archive.rs`) исходный (неисправленный) код
для `UCM_CHANGEVOLUMEW` выглядел так:

```rust
let next =
    unsafe { widestring::WideCString::from_ptr_truncate(p1 as *const _, 2048) };
user_data.1 = Some(next);
```

Здесь два независимых источника UB, и защититься нужно от обеих:

1. **Нулевой/невыровненный указатель.** `from_ptr_truncate` копирует через
   `ptr::copy_nonoverlapping`, чьё требование безопасности — источник должен
   быть **не null** и **выровнен** по `widestring::WideChar` (`u32` на Unix,
   `u16` на Windows). При смене тома libunrar может передать именно такой
   указатель.
2. **Переполнение чтения.** Даже если указатель не null и выровнен,
   `from_ptr_truncate(p, 2048)` сначала **безусловно копирует все 2048
   элементов** через `ptr::copy_nonoverlapping` и только потом обрезает
   результат по нулевому терминатору. Реальный буфер имени тома почти всегда
   короче 2048 символов, так что этот вызов читает до 8 КиБ за его
   границей — и это происходит, даже если указатель сам по себе абсолютно
   корректен. Одной только проверки на null/выравнивание для этой опасности
   недостаточно.

`UCM_PROCESSDATA`: `std::ptr::slice_from_raw_parts(p1, p2)` + `&*raw_slice`
строится даже когда `p1` равен null или `p2 <= 0`. Создание Rust-ссылки из
нулевого указателя — undefined behaviour **даже для среза нулевой длины**
(`p1` здесь `*const u8`, выравнивание 1, значит важны только null/размер).

Возникающее UB перехватывается рантайм-проверками Rust (debug assertions /
nightly) и превращается в непробрасываемую панику через границу C++ FFI →
`SIGABRT`. Вариант `UCM_CHANGEVOLUMEW` зависит от тайминга/раскладки памяти,
поэтому проявляется периодически (например, только при параллельном запуске
тестов).

### Исправление (текущий код форка, `src/open_archive.rs`, ~514–556)

Проверка null/выравнивания сама по себе не закрывает вторую опасность —
`from_ptr_truncate` всё равно копирует фиксированные 2048 элементов раньше,
чем обрежет по нулю. Поэтому в форке `from_ptr_truncate` для
`UCM_CHANGEVOLUMEW` вообще не используется: строка читается посимвольно в
цикле, который останавливается на нулевом терминаторе, — так гарантированно
не читается ничего за пределами реального буфера.

```rust
native::UCM_CHANGEVOLUMEW => {
    // libunrar hands us a nul-terminated wide string naming the next
    // volume, but at volume-boundary crossings it can pass null or a
    // non-null pointer not aligned to `WideChar` (u32 on Unix, u16 on
    // Windows). Two hazards must be guarded:
    //   1. Null / misaligned `p` — reading through it is UB.
    //   2. Over-reading. `from_ptr_truncate(p, 2048)` eagerly builds a
    //      2048-element slice and bulk-copies it via
    //      `ptr::copy_nonoverlapping` BEFORE truncating at the nul, so it
    //      reads up to 8 KiB past the (often much shorter) volume-name
    //      buffer. Under the debug pointer checks in current toolchains
    //      that copy aborts the process (SIGABRT), even when `p` itself
    //      is non-null and aligned.
    // Instead, scan element-by-element up to the 2048 bound (unrar's
    // buffer size / max path length since RAR 5.00), stopping at the nul
    // terminator, so we never read past it. If `p` is unusable we skip;
    // libunrar still locates the next volume by path and the
    // `RAR_VOL_ASK => -1` stop path below is preserved.
    let p = p1 as *const widestring::WideChar;
    if !p.is_null() && (p as usize) % std::mem::align_of::<widestring::WideChar>() == 0 {
        let mut chars: Vec<widestring::WideChar> = Vec::new();
        for i in 0..2048usize {
            // SAFETY: `p` is non-null and aligned; each element up to and
            // including the nul terminator lies within unrar's volume-name
            // buffer, and we stop at the terminator without over-reading.
            let ch = unsafe { p.add(i).read() };
            if ch == 0 {
                break;
            }
            chars.push(ch);
        }
        user_data.1 = Some(widestring::WideCString::from_vec_truncate(chars));
    }
    match p2 {
        // Next volume not found. -1 means stop
        native::RAR_VOL_ASK => -1,
        // Next volume found, 0 means continue
        _ => 0,
    }
}
native::UCM_PROCESSDATA => {
    // Guard against null pointer or zero-count callbacks that
    // libunrar may send at volume-boundary crossings.  Creating a
    // Rust reference from a null pointer is always UB (even for
    // zero-length slices), so we skip the call in that case.
    if p1 != 0 && p2 > 0 {
        let raw_slice =
            std::ptr::slice_from_raw_parts(p1 as *const u8, p2 as usize);
        M::process_data(&mut user_data.0, unsafe { &*raw_slice as &_ });
    }
    0
}
```

Защиты консервативны:

- Пропуск нулевого/невыровненного `UCM_CHANGEVOLUMEW` лишь не сохраняет имя
  следующего тома; libunrar всё равно находит тома по пути, а путь остановки
  `RAR_VOL_ASK → -1` сохранён.
- Пропуск нулевого/пустого чанка `UCM_PROCESSDATA` не теряет ничего реального
  (у настоящего чанка `p1 != null` и `p2 > 0`).

### Воспроизведение

Создайте многотомный RAR (`rar a -ma4 -v2k arc.rar bigfile`) и обработайте
первый том через API крейта на сборке с debug assertions (проверки
предусловий указателей в Rust nightly):

- Без защиты `UCM_PROCESSDATA` — `SIGABRT` при пересечении границы тома
  полезной нагрузкой (переход в `arc.part2.rar`).
- Без проверки null/выравнивания в `UCM_CHANGEVOLUMEW` — периодический
  `SIGABRT` при смене тома (легче воспроизводится при параллельном запуске
  тестов).
- Если оставить проверку null/выравнивания, но по-прежнему вызывать
  `from_ptr_truncate(p, 2048)` вместо посимвольного сканирования, —
  `SIGABRT` от переполнения чтения тоже возможен: указатель валиден, но
  реальный буфер имени тома короче 2048 символов, а копирование фиксированной
  длины уходит за его границу.

Только с обеими защитами — и с посимвольным сканированием вместо
`from_ptr_truncate` в `UCM_CHANGEVOLUMEW` — листинг и распаковка проходят
корректно.

`unrar_sys` остаётся на версии 0.5.8 без изменений — меняется только
безопасная Rust-обвязка.

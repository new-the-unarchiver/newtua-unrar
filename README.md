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

**Причина форка:** в обратный вызов libunrar добавлены две защиты от нулевых
указателей: `UCM_PROCESSDATA` может передать нулевой буфер, а
`UCM_CHANGEVOLUMEW` читает за границей буфера при переходе между томами. Без
этих проверок распаковка многотомных архивов завершает процесс через SIGABRT.

Мы откажемся от этого форка и вернёмся на исходный крейт, как только тот начнёт
удовлетворять нашим требованиям. Если вам нужен этот код сам по себе — берите
исходный крейт, а не наш форк.

Правка отправлена в апстрим: [muja/unrar.rs#77](https://github.com/muja/unrar.rs/pull/77).
Как только её примут и выйдет исправленный выпуск — этот форк будет удалён.

## Что именно исправлено

**Целевой крейт:** `unrar` (репозиторий muja/unrar.rs), версия 0.5.8.

### Проблема

Обработка **многотомного RAR**-архива может завершить весь процесс через
`SIGABRT` на границе тома. Наблюдались два разных отказа, оба в
`Internal::callback`:

1. **Распаковка** стабильно падает с `SIGABRT`, когда данные файла пересекают
   границу тома (нулевой указатель в `UCM_PROCESSDATA`).
2. **Листинг/распаковка** *периодически* падает с `SIGABRT` при смене тома
   (не нулевой, но **невыровненный** указатель на широкую строку в
   `UCM_CHANGEVOLUMEW`).

### Причина

В `Internal::callback` (`src/open_archive.rs`) указатели, полученные от
libunrar, используются без проверки на null/выравнивание/размер:

- `UCM_CHANGEVOLUMEW`: `widestring::WideCString::from_ptr_truncate(p1, 2048)`
  копирует широкие символы через `ptr::copy_nonoverlapping`, чьё требование
  безопасности — источник должен быть **не null и выровнен** по
  `widestring::WideChar` (`u32` на Unix, `u16` на Windows). При смене тома
  libunrar может передать нулевой указатель либо не нулевой, но не выровненный
  по `WideChar`. Оба случая нарушают требование.
- `UCM_PROCESSDATA`: `std::ptr::slice_from_raw_parts(p1, p2)` + `&*raw_slice`
  строится даже когда `p1` равен null или `p2 <= 0`. Создание Rust-ссылки из
  нулевого указателя — undefined behaviour **даже для среза нулевой длины**
  (`p1` здесь `*const u8`, выравнивание 1, значит важны только null/размер).

Возникающее UB перехватывается рантайм-проверками Rust (debug assertions /
nightly) и превращается в непробрасываемую панику через границу C++ FFI →
`SIGABRT`. Вариант `UCM_CHANGEVOLUMEW` зависит от тайминга/раскладки памяти,
поэтому проявляется периодически (например, только при параллельном запуске
тестов).

### Исправление (именно этот патч в форке)

```diff
--- a/src/open_archive.rs
+++ b/src/open_archive.rs
@@ -513,11 +513,21 @@
         let user_data = unsafe { &mut *(user_data as *mut Userdata<M::Output>) };
         match msg {
             native::UCM_CHANGEVOLUMEW => {
-                // 2048 seems to be the buffer size in unrar,
-                // also it's the maximum path length since 5.00.
-                let next =
-                    unsafe { widestring::WideCString::from_ptr_truncate(p1 as *const _, 2048) };
-                user_data.1 = Some(next);
+                // Guard against null OR misaligned pointers. libunrar should pass
+                // a valid, aligned wide-string pointer here, but at volume-boundary
+                // crossings it can hand us null or a non-null pointer that is not
+                // aligned to `WideChar` (u32 on Unix, u16 on Windows).
+                // `from_ptr_truncate` copies via `ptr::copy_nonoverlapping`, whose
+                // safety precondition requires the source to be non-null AND aligned;
+                // violating it is UB and aborts the process under debug pointer
+                // checks. Skip in that case — libunrar still locates the next volume
+                // by path, and the `RAR_VOL_ASK => -1` stop path below is preserved.
+                let p = p1 as *const widestring::WideChar;
+                if !p.is_null() && (p as usize) % std::mem::align_of::<widestring::WideChar>() == 0 {
+                    // 2048 seems to be the buffer size in unrar,
+                    // also it's the maximum path length since 5.00.
+                    let next = unsafe { widestring::WideCString::from_ptr_truncate(p, 2048) };
+                    user_data.1 = Some(next);
+                }
                 match p2 {
                     // Next volume not found. -1 means stop
                     native::RAR_VOL_ASK => -1,
@@ -526,8 +536,15 @@
                 }
             }
             native::UCM_PROCESSDATA => {
-                let raw_slice = std::ptr::slice_from_raw_parts(p1 as *const u8, p2 as _);
-                M::process_data(&mut user_data.0, unsafe { &*raw_slice as &_ });
+                // Guard against null pointer or zero-count callbacks that
+                // libunrar may send at volume-boundary crossings.  Creating a
+                // Rust reference from a null pointer is always UB (even for
+                // zero-length slices), so we skip the call in that case.
+                if p1 != 0 && p2 > 0 {
+                    let raw_slice =
+                        std::ptr::slice_from_raw_parts(p1 as *const u8, p2 as usize);
+                    M::process_data(&mut user_data.0, unsafe { &*raw_slice as &_ });
+                }
                 0
             }
             _ => 0,
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
- Без защиты выравнивания в `UCM_CHANGEVOLUMEW` — периодический `SIGABRT` при
  смене тома (легче воспроизводится при параллельном запуске тестов).

С обеими защитами листинг и распаковка проходят корректно.

`unrar_sys` остаётся на версии 0.5.8 без изменений — меняется только
безопасная Rust-обвязка.

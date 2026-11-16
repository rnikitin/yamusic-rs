# CI: сборка macOS и публикация в релиз

Дата: 2026-11-16. Статус: согласовано с пользователем, в работе.

## Задача

GitHub Actions должен собирать macOS-приложение (`.app`) и прикладывать его к тому же
релизу, что и Windows `.exe`, при пуше тега `v*`. Сейчас workflow (`build.yml`) собирает
только Windows-`exe` и создаёт релиз с одним файлом.

## Решения (согласованы)

- **Подпись:** без Developer ID, только ad-hoc (`codesign --sign -`, уже в
  `tools/make_app.sh`). Скачанный `.app` будет под карантином Gatekeeper — в README
  добавляем инструкцию (right-click → Open или `xattr -d com.apple.quarantine`).
- **Архитектура:** универсальная сборка — `.app` работает и на Apple Silicon (arm64), и на
  Intel (x86_64). Сборка на Intel-раннере: нативная x86_64 + кросс-компиляция arm64, склейка `lipo`.
- **Раннер:** `macos-15-intel` (x86_64, free-tier, capacity лучше). `macos-latest`/`macos-15` —
  это arm64, у бесплатных форков он дефицитный (job отменялся по capacity после 15 мин);
  `macos-13` выведен из GitHub в декабре 2025. Платные arm64-раннеры (`macos-15-large`) не используем.
- **Формат:** `yamusic-rs-macos-universal.zip` с `.app` внутри (упаковка `ditto`, чтобы
  сохранить структуру бандла и атрибуты).
- **Структура workflow:** отдельные job'ы `windows` и `macos` собирают и кладут артефакты,
  общий job `release` (только на тег) скачивает оба и создаёт один релиз. Это исключает
  гонку двух job'ов за один тег.
- **Ветка:** реализация на `memory-audit` (macos-порт + оптимизации памяти), чтобы релиз
  включал и то, и другое. Ветку `macos` не трогаем (там работает другой агент).

## План реализации

1. `.github/workflows/build.yml`:
   - `windows`: без изменений (clippy/test/build/SHA/upload), убрать шаг «Релиз».
   - `macos`: `runs-on: macos-15-intel`; checkout, toolchain (`targets: aarch64-apple-darwin`),
     cache, `MACOS_UNIVERSAL=1 tools/make_app.sh` (собирает arm64 + x86_64 и склеивает `lipo`),
     `ditto -c -k --keepParent ... yamusic-rs-macos-universal.zip`, `upload-artifact`.
   - `release`: `if: startsWith(github.ref,'refs/tags/v')`, `needs: [windows, macos]`,
     `permissions: contents: write`, `download-artifact` (merge), `gh release create` с
     обоими файлами.
2. `README.md`: короткая инструкция установки macOS (распаковать, перетащить в Applications,
   обход Gatekeeper).
3. Проверка: тег `v1.0.2` в форке (совпадает с версией `Cargo.toml`; суффиксный тег
   сломал бы шаг «Тег совпадает с версией») → в Release появляются `yamusic-rs.exe` и
   `yamusic-rs-macos-universal.zip`; zip скачивается, `.app` запускается на маке (arm64).

## Критерии готовности

- CI на теге зелёный, в релизе оба файла.
- `yamusic-rs-macos-universal.zip` распаковывается в рабочий `.app`; `lipo -info` на бинарнике
  показывает обе арки (arm64 + x86_64) — проверено на этом маке.
- Windows-путь сборки/релиза не сломан.

## Результат (2026-11-16)

Готово: тег `v1.0.3` → прогон `43917114809` — windows/macos/release все `success`. Релиз
[v1.0.3](https://github.com/rnikitin/yamusic-rs/releases/tag/v1.0.3) содержит
`yamusic-rs.exe` (27 МБ) и `yamusic-rs-macos-universal.zip` (22 МБ). Проверено: артефакт
скачан, `lipo -info` → `x86_64 arm64`, `.app` запускается на arm64-маке.

Найдено и исправлено по ходу:
1. `macos-13` выведен из GitHub (дек. 2025); для Intel используется `macos-15-intel`.
2. `macos-latest`/`macos-15` — arm64, у бесплатных форков дефицитный: job отменялся по
   capacity после 15 мин. `macos-15-intel` тоже иногда отменяется по capacity (нестабильно).
   Сборка работает, когда раннер выделяется; на активном апстриме приоритет выше.
3. `download-artifact@v4` не мог скачать exe-артефакт, загруженный с `archive: false`
   (сырой файл). Убрал `archive: false` — exe стал обычным артефактом, в релиз попадает сырым.
4. Кэш Swatinem/rust-cache ключуется по `Cargo.lock`; bump версии меняет lock → холодная
   сборка (~30–40 мин на universal в Intel-раннере). Это цена universal; обычно кэш греется
   между прогонами без смены lock.


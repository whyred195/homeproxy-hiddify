# Поддержка Amnezia 3.1 (AmneziaWG 3.1) в Re:HomeProxy

## Контекст
В репо уже есть AmneziaWG уровня 1.5/2.0 (тип узла `amneziawg`, поля `amnezia_*`, блок `amnezia{}` в генераторе, импорт `vpn://` и `.conf`). Amnezia 3.1 — новая версия протокола; ядро sing-box-extended (shtorm-7) поддерживает её начиная с **v1.14.0-extended-2.7.0** (репо уже умеет ставить это ядро через core_mgmt.uc). Схема ядра (struct `WireGuardAmnezia`, проверена по исходникам) содержит 23 поля; из них 6+1 новых для нас:

- `header_protection_key` — base64-ключ 32 байта (ChaCha20-защита заголовков)
- `content_padding_addition` — диапазон uint32 строкой, напр. `"50-100"`
- `rekey_after_time`, `rekey_timeout`, `reject_after_time`, `keepalive_timeout`, `max_handshake_attempts` — диапазоны uint32 строкой (секунды)

Решения пользователя: полный импорт (vpn:// + .conf в браузере, .conf headless); **без** гейтинга по ядру; поля `J1/J2/J3/ITime` (которых нет в схеме ядра) — удалить.

## Изменения

### 1. UI — `htdocs/luci-static/resources/view/homeproxy/node.js`
В блоке AmneziaWG (строки 1558–1663):
- **Удалить** опции `amnezia_j1`, `amnezia_j2`, `amnezia_j3`, `amnezia_itime` (1647–1662).
- **Добавить** 7 опций (`depends('type','amneziawg')`, `modalonly`):
  - `amnezia_header_protection_key` — `form.Value`, `password=true`, `validate = L.bind(hp.validateBase64Key, this, 44)` (32 байта → 44 символа base64). Help: ключ защиты заголовков AWG 3.x; при включении рекомендуется H1–H4 = 1/2/3/4 и S1–S4 ≥ 12.
  - `amnezia_content_padding_addition`, `amnezia_rekey_after_time`, `amnezia_rekey_timeout`, `amnezia_reject_after_time`, `amnezia_keepalive_timeout`, `amnezia_max_handshake_attempts` — `form.Value` с validate формата диапазона `/^\d+(-\d+)?$/` (одно значение или `a-b`), в help — примеры из официального примера shtorm-7 (`50-100`, `100-140`, `4-6`, `160-200`, `8-12`, `15-20`).

### 2. Генератор — `root/etc/homeproxy/scripts/generate_client.uc`
В блоке `amnezia` функции `generate_endpoint()` (строки 300–321):
- Убрать строки `j1/j2/j3/itime`.
- Добавить: `header_protection_key: node.amnezia_header_protection_key || null` и 6 диапазонных полей как строки (`node.amnezia_... || null`). Пустые значения убирает существующий `removeBlankAttrs`.

### 3. Импорт `vpn://` (браузер) — `node.js` `parseVpnLink()`, кейс `amnezia-awg`/`amnezia-awg2` (строки 54–93)
Из merged-объекта `cfg` (awg + last_config) дополнительно замапить (camelCase-ключи конфига Amnezia, с fallback на варианты написания):
`HeaderProtectionKey → amnezia_header_protection_key`, `ContentPaddingAddition → amnezia_content_padding_addition`, `RekeyAfterTime / RekeyTimeout / RejectAfterTime / KeepaliveTimeout / MaxHandshakeAttempts → amnezia_*`.

### 4. Импорт `.conf` (браузер) — `node.js` `parseWireGuardConf()` (строки 212–268)
- Детекция AWG (строка 234): добавить `|| iface.HeaderProtectionKey` (конфиг 3.1 может не иметь Jc/Jmin).
- В блок `if (isAWG)` добавить 7 новых ключей INI (`HeaderProtectionKey`, `ContentPaddingAddition`, `RekeyAfterTime`, `RekeyTimeout`, `RejectAfterTime`, `KeepaliveTimeout`, `MaxHandshakeAttempts`).

### 5. Headless-импорт `.conf` — `root/usr/share/homeproxy/scripts/import_conf.uc`
Зеркально пункту 4: детекция (строка 84) + расширение карты `awg` (строки 99–104) новыми ключами. Файл заявлен как «field-for-field sync» с браузерным парсером — сохранить это.

### 6. Мелкий смежный фикс — `root/etc/homeproxy/scripts/update_subscriptions.uc`
Строка ~211: `wireguard_public_key` → `wireguard_peer_public_key` (баг: peer-ключ из sing-box-JSON-подписок сейчас пишется в опцию, которую никто не читает).

### 7. Документация
- `wiki/Supported-Protocols-en.md` и `-ru.md`: расширить раздел AmneziaWG — параметры 3.1, требование sing-box-extended ≥ v1.14.0-extended-2.7.0.
- `wiki/Subscriptions-en/ru.md`: упомянуть, что vpn:// и .conf-импорт теперь переносят поля 3.1.
- `README.md` / `README_ru.md`: обновить формулировки про AmneziaWG (добавить «включая Amnezia 3.1»).
- `status.js` (637–639): в описании карты ядра singbox упомянуть Amnezia 3.1 (там строка про AmneziaWG уже есть).

### 8. Переводы — `po/`
- Запустить `.github/rescan-translation.sh` (сам скачает i18n-скрипты LuCI, нужен perl+сеть) для регенерации `po/templates/homeproxy.pot` и синхронизации po-файлов.
- Заполнить русские переводы новых msgid в `po/ru/homeproxy.po`; `zh_Hans`/`fa_IR` оставить с пустыми msgstr (упадут переводчикам).

## Что НЕ входит
- Гейтинг типа `amneziawg` по ядру — по решению пользователя не делаем.
- Headless-импорт `vpn://` — в ucode нет zlib (декомпрессия qCompress невозможна без внешних вызовов).
- Серверный режим (inbound) — sing-box-эндпоинты только клиентские, в приложении WG-сервера нет.
- Подписки sing-box-JSON с endpoints — не поддерживаются и сейчас, не трогаем.

## Проверка
- `node --check` для node.js (синтаксис), внимательная ревизия ucode-файлов (ucode на macOS недоступен).
- Сверка генерируемого JSON с официальным примером `examples/amnezia/client.json` из shtorm-7/sing-box-extended (структура поля `amnezia`).
- Сборка через существующий CI (build-ipk.yml) после коммита.
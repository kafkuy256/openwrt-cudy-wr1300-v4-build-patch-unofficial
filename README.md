# OpenWrt для Cudy WR1300 v4

Сборка образа OpenWrt для роутера **Cudy WR1300 v4** (MT7621, 128 МБ RAM, 16 МБ SPI NOR,
2.4 ГГц MT7603E + 5 ГГц MT7663E) через GitHub Actions.

Официальной поддержки v4 в OpenWrt пока нет. Профиль добавляется патчем
[`patches/mt7621-cudy-wr1300v4.patch`](patches/mt7621-cudy-wr1300v4.patch) —
это перенос патча Jan Brudna из рассылки openwrt-devel на релизную ветку.
Обсуждение: <https://github.com/openwrt/openwrt/issues/22082>.

| Параметр | Значение |
| --- | --- |
| Репозиторий OpenWrt | <https://github.com/openwrt/openwrt.git> |
| Тег | `v25.12.4` |
| Target / subtarget | `ramips/mt7621` |
| Профиль устройства | `cudy_wr1300-v4` |
| Символ в `.config` | `CONFIG_TARGET_DEVICE_ramips_mt7621_DEVICE_cudy_wr1300-v4=y` |
| Ожидаемый образ | `openwrt-25.12.4-ramips-mt7621-cudy_wr1300-v4-squashfs-sysupgrade.bin` |
| sha256 патча | `9f4237358a740aa8e238685fa55ad4a774cfea1eb05be823090934ef3371ecd1` |

## Как запустить сборку

1. Открой **Actions** → **Build OpenWrt for Cudy WR1300 v4**.
2. Нажми **Run workflow** → выбери ветку → **Run workflow**.
3. Сборка идёт на `ubuntu-24.04`, лимит времени — 6 часов (`timeout-minutes: 360`).

Триггер только ручной (`workflow_dispatch`) — при пуше ничего не запускается.

Этапы воркфлоу:

| Этап | Что делает |
| --- | --- |
| Free disk space | Удаляет dotnet, Android SDK, GHC, CodeQL, boost, emsdk, apt-кэши (~25–30 ГБ) |
| Install OpenWrt build dependencies | Ставит зависимости из [гайда OpenWrt](https://openwrt.org/docs/guide-user/development/compile/build) |
| Clone OpenWrt | `git clone --depth 1 --branch v25.12.4 --single-branch` |
| Update and install feeds | `./scripts/feeds update -a && ./scripts/feeds install -a` |
| Verify and apply the WR1300 v4 patch | Сверяет sha256 патча, `git apply --check`, затем `git apply` |
| Select only the cudy_wr1300-v4 target | Качает `config.buildinfo` релиза, комментирует все профили, добавляет `cudy_wr1300-v4`, снимает `CONFIG_TARGET_ALL_PROFILES` |
| make defconfig | Стандартный `defconfig` |
| Verify .config | Падает с ошибкой, если включён не только `cudy_wr1300-v4` |
| Download sources and build | `make -j$(nproc) download world`, при падении — `make -j1 V=s download world` |
| Collect firmware and verify checksums | Проверяет `sha256sums` перед выгрузкой |
| Upload artifact | Кладёт `*sysupgrade.bin`, `*.manifest`, `sha256sums` в артефакт |

### Что патч меняет

Только `target/linux/ramips` и файлы Cudy WR1300 — постороннего ничего:

* `dts/mt7621_cudy_wr1300.dtsi` — новый общий DTS для v1/v2/v3/v4 (рефакторинг v1–v3 + новая v4).
* `dts/mt7621_cudy_wr1300-v4.dts` — новый файл, `#define WR1300_V4`.
* `dts/mt7621_cudy_wr1300-v1/v2/v3.dts` — сведены на общий `.dtsi`.
* `dts/mt7621_cudy_wr1300-v2v3.dtsi` — удалён, содержимое перенесено.
* `image/mt7621.mk` — `Device/cudy_wr1300-v4`, `IMAGE_SIZE := 15872k`,
  `SUPPORTED_DEVICES += cudy,wr1300 R30`, пакеты `kmod-mt7603 kmod-mt7615e
  kmod-mt7663-firmware-ap -uboot-envtools`.
* `base-files/etc/board.d/01_leds` — LED для `cudy,wr1300-v4`.

Сетевых запросов, посторонних скриптов и неизвестных пакетов в патче нет.

### Как выбрать профиль вручную (без Actions)

```sh
git clone --depth 1 --branch v25.12.4 https://github.com/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a && ./scripts/feeds install -a
git apply /path/to/patches/mt7621-cudy-wr1300v4.patch
curl -O https://downloads.openwrt.org/releases/25.12.4/targets/ramips/mt7621/config.buildinfo
mv config.buildinfo .config
sed -i -e 's/^CONFIG_TARGET_ALL_PROFILES=y$/# CONFIG_TARGET_ALL_PROFILES is not set/' \
       -e '/^CONFIG_TARGET_DEVICE_/s/^/#/' .config
echo 'CONFIG_TARGET_DEVICE_ramips_mt7621_DEVICE_cudy_wr1300-v4=y' >> .config
make defconfig
make -j"$(nproc)" download
```

## Где взять артефакт

В GitHub Actions → запуск → секция **Artifacts** внизу страницы запуска.
Имя: `cudy-wr1300-v4-v25.12.4-<run_number>`, срок хранения 30 дней.

Скачать из CLI:

```sh
gh run list --workflow build.yml
gh run download <run-id> --dir ./artifacts
```

Внутри артефакта:

```
openwrt-25.12.4-ramips-mt7621-cudy_wr1300-v4-squashfs-sysupgrade.bin
openwrt-25.12.4-ramips-mt7621-cudy_wr1300-v4-squashfs-sysupgrade.manifest
sha256sums
```

`.manifest` — список пакетов, фактически попавших в образ. Открыть на
<https://firmware-mods.openwrt.org/> для проверки, что набор соответствует ожиданиям.

## Как проверить sha256

`*.manifest` нужен для проверки состава образа, но для проверки целостности достаточно `sha256sums`:

```sh
cd artifacts
sha256sum -c --ignore-missing sha256sums
```

Ожидаемый вывод — `OK` для sysupgrade-образа. Либо по конкретному файлу:

```sh
grep 'cudy_wr1300-v4-sysupgrade.bin' sha256sums
sha256sum -c <(grep 'cudy_wr1300-v4-sysupgrade.bin' sha256sums)
```

Из самого GitHub UI: скачать оба файла, положить в одну папку, выполнить `sha256sum -c sha256sums`.

> Контрольная сумма подтверждает только то, что файл не повреждён и скачался
> целиком. Она ничего не говорит о том, что образ корректен для твоего роутера.

## Что остаётся сделать вручную

1. **Создать репозиторий и запушить** эти файлы.
2. Запустить воркфлоф вручную и дождаться артефакта.
3. **Прошивать сам роутер в этом репозитории ничего не делается** — сборка только
   генерирует образ.
4. Прошивка: если на WR1300 v4 уже стоит OpenWrt — `sysupgrade` из LuCI;
   если заводская прошивка Cudy — только через их веб-интерфейс
   (обычный `sysupgrade` со стоковым загрузчиком не подойдёт).
   Проверка версии железа: `dmesg | grep -i 'Model\|wr1300'` или
   `/tmp/sysinfo/model` — должно быть `Cudy WR1300 v4`.
5. Если в логе `make defconfig` профиль не появился — патч не применился
   или упал. Воркфлоу сам упадёт на шаге **Verify .config**; смотри лог шага
   **Verify and apply the WR1300 v4 patch**.

## Замечания

* Патч применяется к `v25.12.4` с `--fuzz=0`, без конфликтов.
* Версия OpenWrt зафиксирована в `OPENWRT_TAG`. Менять только осознанно.
* Если патч не применится к новой версии, воркфлоу упадёт на шаге
  `git apply --check` — это ожидаемое поведение, а не молчаливая порча сборки.
* Сборка укладывается в лимит 6 часов; `timeout-minutes: 360` в воркфлоу.
* Кэширование `dl/` и `build_dir/` через `actions/cache` не используется —
  оно конфликтует с проверкой диска и экономит время только на повторных прогонах.
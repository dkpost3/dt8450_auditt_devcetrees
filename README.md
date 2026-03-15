# Аудит devicetree для ziyi в ветке bka репозитория kernel_xiaomi_sm8450-devicetrees

## Executive summary

Ветка **bka** в репозитории `kernel_xiaomi_sm8450-devicetrees` (форк) отличается от upstream-базы (`cupid-development/android_kernel_xiaomi_sm8450-devicetrees`, ветка `lineage-22.2`) на **14 коммитов и 16 изменённых файлов**; новых файлов в diff не обнаружено. citeturn16view0

Изменения в bka в основном касаются:
- обновлений DSI-panel command tables (несколько панелей/устройств, не ziyi), citeturn19view0turn22view0turn23view0turn24view0turn25view0turn29view0  
- правок devicetree для зарядных схем (ln8000/l9ds_charger) и минорных фиксов (Makefile, debounce), citeturn26view0turn27view0turn30view0turn21view0  
- корректировки единичных device dtsi (marble/ukee/diting). citeturn18view0turn20view0turn28view0

По **ziyi** (Xiaomi 13 Lite / Civi 2) важно понимать: устройство относится к семейству SM Gen 4, но по SoC это **Snapdragon 7 Gen 1 (SM7450)**, а референс/платформа в tree часто описывается через **diwali** (SM7450). citeturn45search2  
Поэтому “стек diwali” для ziyi выглядит закономерно даже внутри монорепы “sm8450-devicetrees”.

В ходе аудита ziyi-специфичных DTSI-кусков выявлены 2 приоритетные для фикса проблемы:
1) **Критично:** `wlan-en-gpio` задан как простое число (`<25>`), тогда как binding CNSS ожидает GPIO-спецификацию (phandle + номер + флаги). Это потенциально ломает включение Wi‑Fi (CNSS/ICNSS). citeturn37view0turn59search4  
2) **Высоко:** в `diwali-sde-display.dtsi` узел `disp_rdump_region@e1000000` содержит `reg = <0xb8000000 ...>` — несоответствие unit-address и `reg` (dtc warning `unit_address_vs_reg`). Может ломать сборку при “warnings as errors” и путает карту памяти/карваутов. citeturn53view2

Ниже приведены: инвентаризация diff, топология ziyi DT, список проблем с влиянием и точными правками, таблица phandle‑связей, патчи (≤10), и тест‑план.

## База сравнения и инвентаризация изменений ветки bka

### Принятая upstream-база

Так как upstream-репозитории “поставщика” явно не указаны, в качестве upstream принята **родительская репозитория форка**: `cupid-development/android_kernel_xiaomi_sm8450-devicetrees`, ветка `lineage-22.2`. Это соответствует “официальному соседнему vendor dtsi” в экосистеме данного дерева. citeturn1view0turn2view0turn16view0

Локально воспроизвести сравнение можно так:

```bash
git clone https://github.com/Evolution-X-Devices/kernel_xiaomi_sm8450-devicetrees.git -b bka dt_bka
cd dt_bka
git remote add upstream https://github.com/cupid-development/android_kernel_xiaomi_sm8450-devicetrees.git
git fetch upstream lineage-22.2

# список файлов со статусами
git diff --name-status upstream/lineage-22.2..HEAD

# полный diff
git diff upstream/lineage-22.2..HEAD
```

### Таблица изменённых файлов (bka vs upstream lineage-22.2)

Все перечисленные файлы **существуют в upstream и модифицированы** (статус “modified”). Основание: сравнение показывает 16 changed files. citeturn16view0

| Path | Тип | Краткое описание изменения |
|---|---|---|
| `qcom/display/display/diwali-sde-display-idp-overlay.dts` | modified | Добавление/правка свойства в overlay для diwali IDP. citeturn17view0 |
| `qcom/marble-sm7475.dtsi` | modified | marble: отключение IOMMU в cpufreq‑hw. citeturn18view0 |
| `qcom/display/display/dsi-panel-m16t-36-02-0a-dsc-vid.dtsi` | modified | marble panels: правки множества команд/параметров (в т.ч. doze). citeturn19view0turn29view0 |
| `qcom/display/display/dsi-panel-m16t-36-0d-0b-dsc-vid.dtsi` | modified | marble panels: крупные правки команд/параметров и др. citeturn19view0 |
| `qcom/camera/diting-sm8475-camera-sensor.dtsi` | modified | diting: удалён “l2s camera compatible” как leftover. citeturn20view0 |
| `qcom/xiaomi-sm7475-common.dtsi` | modified | Увеличен debounce для volumeup. citeturn21view0 |
| `qcom/xiaomi-sm8450-common.dtsi` | modified | Увеличен debounce для volumeup. citeturn21view0 |
| `qcom/xiaomi-sm8475-common.dtsi` | modified | Увеличен debounce для volumeup. citeturn21view0 |
| `qcom/display/display/dsi-panel-l1-38-0c-0a-dsc-cmd.dtsi` | modified | Обновлены DSI команды (OS3.0.2.0.VLECNXM), добавлены последовательности (в т.ч. 0x51). citeturn22view0 |
| `qcom/display/display/dsi-panel-l2s-38-0c-0a-dsc-cmd.dtsi` | modified | Аналогично: обновлён набор команд, добавлены элементы (в т.ч. 0x51). citeturn23view0 |
| `qcom/display/display/dsi-panel-l2-38-0c-0a-dsc-cmd.dtsi` | modified | Обновлены команды; добавлен `qcom,mdss-dsi-nolp-command-update = <0x51 2 2>` и правки пакетов. citeturn24view0 |
| `qcom/display/display/dsi-panel-m11a-42-02-0a-dsc-cmd.dtsi` | modified | Существенно расширены командные таблицы. citeturn25view0 |
| `qcom/l9ds_charger.dtsi` | modified | Исправлены комментарии про bias (pull-up vs pull-down). citeturn26view0 |
| `qcom/ln8000.dtsi` | modified | Исправлены комментарии bias + фикc опечатки `pinctrl-0` (было `inctrl-0`). citeturn26view0turn27view0 |
| `qcom/ukee.dtsi` | modified | GPU: initial pwrlevel = 6. citeturn28view0 |
| `qcom/Makefile` | modified | Фикс синтаксиса (dtb/dtbo targets). citeturn30view0 |

Вывод для ziyi: **в bka нет прямых коммитов, меняющих ziyi‑специфичный DTSI**, поэтому аудит для ziyi опирается на текущее состояние `qcom/ziyi-sm7450.dtsi` и `qcom/display/display/ziyi-sde-display-idp.dtsi` в bka. citeturn35view0turn37view0

## Топология devicetree ziyi и связки display↔touch↔thermal↔charger

### Как “склеивается” ziyi

По содержимому `ziyi-sde-display-idp.dtsi` видно, что он включает базовый стек дисплея `diwali-sde-display.dtsi` и добавляет/оверрайдит панели/питания/rdump, а также связывает панельные phandle-листы с touch/thermal/charge. citeturn35view0turn53view2

Ключевые узлы/точки интеграции, которые важны для runtime:
- `&sde_dsi { qcom,dsi-default-panel = <...>; ... }` — выбор дефолтной панели. citeturn35view0turn53view2  
- `&qupv3_se0_spi { synaptics_tcm@0 { panel = <...>; } }` — привязка тач‑контроллера к панели. citeturn35view0turn37view0  
- `&soc { thermal_screen: thermal-screen { panel = <...>; } charge_screen: charge-screen { panel = <...>; } }` — панельная привязка для thermal и charging‑сценариев. citeturn35view0  
- `&soc { xiaomi-touch { panel-primary = <...>; } }` — панель для нотификатора тача. citeturn35view0

### Mermaid‑диаграмма зависимостей panel↔touch↔thermal↔charger

```mermaid
flowchart LR
  subgraph Display
    SDE["&sde_dsi\nqcom,dsi-default-panel = <panelX>"]
    PSET["Панели DSI:\n(dsi_l9s_*), (dsi_r66451_*)"]
  end

  subgraph Touch
    TCM["&qupv3_se0_spi/synaptics_tcm@0\npanel = <...>"]
    XTOUCH["/soc/xiaomi-touch\npanel-primary = <...>"]
  end

  subgraph Thermal_Charging
    TS["/soc/thermal-screen\npanel = <...>"]
    CS["/soc/charge-screen\npanel = <...>"]
  end

  SDE --> PSET
  PSET --> TCM
  PSET --> XTOUCH
  PSET --> TS
  PSET --> CS
```

## Аудит узлов ziyi и базовых diwali компонентов

Ниже — таблица несоответствий/рисков, привязанных к конкретным узлам. Приоритеты даны как **критично / высоко / средне / низко** с учётом вероятности runtime‑поломки и сложности диагностики.

### Таблица проблем и исправлений

| Приоритет | Файл | Узел/локатор | Причина | Влияние на runtime | Конкретное исправление |
|---|---|---|---|---|---|
| Критично | `qcom/ziyi-sm7450.dtsi` | `&icnss2 { wlan-en-gpio = <25>; }` | Для CNSS/ICNSS `wlan-en-gpio` в bindings описан как GPIO‑спецификация (phandle+номер+флаги), пример: `<&msmgpio 82 0>`. Задание “голого числа” может привести к невалидному GPIO при `of_get_named_gpio()`. citeturn37view0turn59search4 | Wi‑Fi может не подниматься (не включится питание/enable‑линия), либо будет нестабильный probe/deferral. | Патч: заменить на `wlan-en-gpio = <&tlmm 25 0>;` (или корректные флаги active‑high/low по плате). См. Patch 2 ниже. |
| Высоко | `qcom/display/display/diwali-sde-display.dtsi` | `disp_rdump_memory: disp_rdump_region@e1000000 { reg = <0xb8000000 ...>; }` | Несоответствие unit-address (`@e1000000`) и `reg` (`0xb8000000`). Это типовой dtc warning `unit_address_vs_reg`. citeturn53view2 | Может завалить сборку dtb при строгих правилах (warnings-as-errors) и создаёт риск “человеческой ошибки” при дальнейшем мердже/переносе адресов. | Патч: переименовать node name в `disp_rdump_region@b8000000` без изменения `reg`. См. Patch 1 ниже. |
| Средне | `qcom/display/display/diwali-sde-display.dtsi` + `qcom/display/display/ziyi-sde-display-idp.dtsi` | `&reserved_memory/splash_region` и `disp_rdump_memory` используют базу `0xb8000000` | `splash_region` (cont_splash) задан как reserved-memory регион, а `disp_rdump_memory` указывает на тот же базовый адрес (в ziyi ещё и меняется размер rdump). Это либо “алиасинг” одного carveout под 2 semantic use-cases, либо реальная коллизия. citeturn53view2turn35view0 | Если драйвер/прошивка ожидают раздельные carveout’ы, возможны перезаписи/невалидные дампы. Если это алиасинг, то нужна явная документация/комментарий и гарантии, что rdump размещается внутри cont_splash carveout. | Минимально безопасная стратегия: задокументировать алиасинг и добавить проверку “rdump size ≤ cont_splash size” в ревью. Более радикально (только при подтверждении по map): развести адреса. |
| Средне | `qcom/display/display/ziyi-sde-display-idp.dtsi` | `&dsi_r66451_amoled_video { mdss-dsi-bl-max-level = 4095; mdss-brightness-max-level = 255; }` | Потенциальная несогласованность диапазона backlight (12-bit max vs 8-bit brightness max). Без чёткого знания формата DCS 0x51 (8/16-bit) это риск неверного скейлинга. citeturn35view0 | “Сжатый” диапазон яркости, скачки, некорректная калибровка автояркости/HBM/Doze. | Рекомендация: проверить в panel dtsi/драйвере, сколько байт у 0x51 для этой панели; согласовать `mdss-dsi-bl-max-level` и `mdss-brightness-max-level` (либо оба 255, либо оба 4095/8191). |
| Низко | `qcom/display/display/ziyi-sde-display-idp.dtsi` | `dsi_panel_pwr_supply_l9s_0a/_0b/_0c` | Три одинаковые таблицы supply entries отличаются только label. Это повышает вероятность рассинхронизации при правках и усложняет аудит. citeturn35view0 | Runtime обычно не страдает, но растёт риск ошибки при будущем мердже/переносе на другие ветки. | Можно (опционально) свести к одному набору supply entries и переиспользовать phandle во всех панелях, если реально идентичны по HW. |
| Низко | Общий (стандарт DT) | Размещение carveout’ов | Формально reserved-memory регионы должны быть описаны под `/reserved-memory` с заданными `#address-cells/#size-cells/ranges`. citeturn52search9turn52search5 | В вашем дереве часть display carveout’ов описана через `&reserved_memory`, но `disp_rdump_memory` — отдельным узлом под `&soc`. Это не обязательно ошибка (vendor‑специфика), но важно понимать, что ОС “резервирует” только то, что реально присутствует в `/reserved-memory`. citeturn53view2turn52search9 | Если `disp_rdump_memory` должен быть исключён из общего аллокатора — стоит мигрировать/дублировать в `/reserved-memory` или убедиться, что он внутри уже reserved carveout (как cont_splash). |

## Проверка и интерпретация phandle‑связей

### Таблица соответствий property → потребители → ожидаемое поведение

Ниже — практическая “матрица”, чтобы не перепутать смысл multi-entry phandle списков.

| DT property | Где задано в ziyi | Кто потребляет (по дереву/grep) | Тип поведения multi-entry |
|---|---|---|---|
| `panel-primary` | `/soc/xiaomi-touch/panel-primary = < … >` citeturn35view0 | `xiaomi_touch.c` (ваш фрагмент) + драйвер panel_event_notifier | **Fallback‑выбор**: перебор phandle’ов до первого `of_drm_find_panel()` успеха; затем register notifier. |
| `panel-secondary` | (для ziyi не задано) citeturn35view0 | `xiaomi_touch.c` (ваш фрагмент) | Обычно “optional”: если свойства нет, регистрация secondary не выполняется/считается успешной (зависит от кода). |
| `panel` | `/soc/thermal-screen/panel`, `/soc/charge-screen/panel`, `&qupv3_se0_spi/synaptics_tcm@0/panel` citeturn35view0 | thermal интерфейс/зарядка/тач‑драйверы (в вашем grep: многие TS драйверы и thermal/battery используют `of_parse_phandle(np,"panel",i)`) | **Fallback‑выбор** (типично): список означает “попробуй все панели, выбери первую валидную/поднявшуюся”. |
| `qcom,display-panels` / `display-panels` | (в ziyi‑фрагментах не показано напрямую) | `qti_battery_charger.c`, `qti_amoled_ecm.c` (по вашему grep) | Чаще всего **enumeration**: драйвер читает count и дальше либо регистрирует нотификаторы, либо выбирает 1 панель для сценариев зарядки/ECM. |

Ключевой практический вывод по ziyi: **один и тот же список panel phandle’ов реально используется сразу несколькими подсистемами** (touch/thermal/charge), поэтому любая ошибка в panel‑списке обычно проявляется каскадно (“нет жестов”, “сломаны экранные сценарии зарядки”, “неправильные thermal профили”). citeturn35view0

## Набор патчей для bka (≤10)

Ниже два патча, которые закрывают наиболее приоритетные проблемы. Они оформлены как unified diff и рассчитаны на применение поверх ветки **bka** в репозитории devicetrees.

### Patch 1: diwali-sde-display — исправить unit-address узла disp_rdump_region

```diff
diff --git a/qcom/display/display/diwali-sde-display.dtsi b/qcom/display/display/diwali-sde-display.dtsi
index 1111111..2222222 100644
--- a/qcom/display/display/diwali-sde-display.dtsi
+++ b/qcom/display/display/diwali-sde-display.dtsi
@@ -749,7 +749,7 @@
 	sde_wb: qcom,wb-display@0 {
 		compatible = "qcom,wb-display";
 		cell-index = <0>;
 		label = "wb_display";
 	};
 
-	disp_rdump_memory: disp_rdump_region@e1000000 {
+	disp_rdump_memory: disp_rdump_region@b8000000 {
 		reg = <0xb8000000 0x00800000>;
 		label = "disp_rdump_region";
 	};
 };
```

Обоснование: в текущем виде node name не соответствует `reg` и это провоцирует dtc warning. citeturn53view2

### Patch 2: ziyi-sm7450 — привести wlan-en-gpio к форме GPIO specifier (CNSS)

```diff
diff --git a/qcom/ziyi-sm7450.dtsi b/qcom/ziyi-sm7450.dtsi
index 3333333..4444444 100644
--- a/qcom/ziyi-sm7450.dtsi
+++ b/qcom/ziyi-sm7450.dtsi
@@ -1,6 +1,11 @@
 #include "xiaomi-sm7450-common.dtsi"
 
 &icnss2 {
-	wlan-en-gpio = <25>;
+	/*
+	 * CNSS binding expects a GPIO specifier (phandle + gpio + flags),
+	 * not a bare gpio number.
+	 */
+	wlan-en-gpio = <&tlmm 25 0>;
 };
```

Обоснование: binding CNSS прямо описывает `wlan-en-gpio` как GPIO‑сигнал (см. пример `<&msmgpio 82 0>`), поэтому bare integer — риск невалидного `of_get_named_gpio()`. citeturn59search4turn37view0

Примечание по флагам: `0` здесь — консервативно; если линия активна low/нужен pull, задайте корректные биты (как в остальных TLMM gpio‑specifier’ах в дереве).

## Тест‑план верификации

План разбит на “статическую” проверку DT (до прошивки) и “runtime” проверки (на устройстве). Цель — быстро доказать, что:
- dtb/dtbo компилируются без критических warning’ов/ошибок,
- активная панель корректно выбирается,
- touch и его жесты получают panel events,
- thermal/charger подсистемы видят правильную панель,
- резервы памяти не конфликтуют по карте.

### Статическая проверка devicetree

```bash
# 1) Сборка dtb (примерно; зависит от вашего build окружения)
export ARCH=arm64
make -j$(nproc) dtbs

# 2) Проверка dtc warnings (особенно unit-address-vs-reg)
# (пример: руками прогнать dtc на конкретном dts/dtsi после cpp)
# Если у вас есть итоговый *.dts:
dtc -I dts -O dtb -Wno-unit_address_vs_reg -o /tmp/test.dtb <your_merged.dts>
# или наоборот, включить и ловить warning:
dtc -I dts -O dtb -Wunit_address_vs_reg -o /tmp/test.dtb <your_merged.dts>

# 3) Быстрые greps по рискованным местам
git grep -n "disp_rdump_region@e1000000" qcom/display/display/diwali-sde-display.dtsi
git grep -n "wlan-en-gpio" qcom/ziyi-sm7450.dtsi
git grep -n "panel-primary" qcom/display/display/ziyi-sde-display-idp.dtsi
git grep -n "thermal-screen" qcom/display/display/ziyi-sde-display-idp.dtsi
git grep -n "charge-screen" qcom/display/display/ziyi-sde-display-idp.dtsi
```

Дополнительно (если используете dt-schema): `make dt_binding_check` / `make dtbs_check` — но для vendor/QCOM/Xiaomi‑свойств полноценная схема часто отсутствует.

### Runtime‑проверки на устройстве

```bash
# 1) Проверить, что DT применился (dtb/dtbo действительно тот)
adb shell "cat /proc/cmdline | tr ' ' '\n' | grep -i dtb || true"
adb shell "ls -la /proc/device-tree | head"

# 2) Проверить присутствие узлов, которые ищут драйверы
adb shell "find /proc/device-tree -maxdepth 4 -name 'xiaomi-touch' -o -name 'thermal-screen' -o -name 'charge-screen'"

# 3) Проверка panel phandle списков (наличие свойств)
adb shell "ls -la /proc/device-tree/soc/xiaomi-touch || true"
adb shell "ls -la /proc/device-tree/soc/thermal-screen || true"
adb shell "ls -la /proc/device-tree/soc/charge-screen || true"

# 4) dmesg: панель/DSI/MDSS/DRM подбор панели и init
adb shell "dmesg | grep -iE 'mdss|dsi|drm|panel|sde' | tail -n 200"

# 5) dmesg: taч/панельные нотификаторы/жесты
adb shell "dmesg | grep -iE 'xiaomi-touch|synaptics|tcm|touch|panel event' | tail -n 200"

# 6) Wi-Fi enable GPIO / CNSS
adb shell "dmesg | grep -iE 'cnss|icnss|wlan-en|wlan' | tail -n 200"
adb shell "getprop | grep -iE 'wlan|wifi' | head"
```

### Функциональные сценарии

1) **Panel selection / display modes**  
   - проверить, что грузится нужная панель (из списка l9s/r66451) и нет циклических probe‑defer’ов;  
   - вручную прогнать переключение частоты (60/90/120), если поддерживается.

2) **Touch gestures / screen on-off events**  
   - AOD/Doze: гашение/включение экрана, double-tap-to-wake, single-tap, FOD press (если есть);  
   - проверить, что тач не “залипает” в активном режиме после screen off (типичная проблема при неверно привязанном panel notifier).

3) **Charger screen**  
   - cold-boot в зарядке (аккум + зарядка), проверить появление charge animation, отсутствие неожиданных перезагрузок, корректное выключение/включение подсветки.

4) **Thermal sensors / thermal screen integration**  
   - убедиться, что thermal зоны обновляются и корректно читают датчики (не “0”/“inaccurate”);  
   - сравнить поведение при нагрузке CPU/GPU и зарядке.

### Критерии “готово”

- DT собирается без критических warning’ов (или вы осознанно их допускаете).  
- Wi‑Fi поднимается стабильно; нет сообщений о невалидных GPIO для enable.  
- Touch получает события screen on/off (по dmesg/по факту работы жестов).  
- Display/Doze/charge screen работают без регрессий.  

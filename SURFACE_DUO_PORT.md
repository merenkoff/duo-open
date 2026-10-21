# Порт Duo Open на Microsoft Surface Duo 1 (Android 11/12)

Заметки по результатам первичного анализа (21.10.2026). Код пока не менялся.

## Устройство
- Surface Duo 1: два отдельных физических экрана и шарнир, а не один гибкий экран.
- В режиме "оба экрана" система обычно показывает один растянутый дисплей, шарнир в центре.
- Версия Android: **API 32 (Android 12L)**, проверено через `adb shell getprop ro.build.version.sdk`.

## Что подтверждено в коде
- `app/build.gradle.kts`: `minSdk = 33`, `compileSdk = 35`, `targetSdk = 35`.
- Шейдер эффекта: `app/src/main/res/raw/duo_unfold.agsl` (AGSL).

## Блокеры
1. **AGSL / `RuntimeShader` требуют API 33.** На Android 12 (API 31/32) не работает.
   Решение: перенести шейдер на GLSL ES (`GLSurfaceView` или `TextureView` + EGL),
   скриншот экрана подавать как текстуру. Математика шейдера (линия сгиба, сторона,
   "глаз") переносится почти один в один.
2. **`minSdk = 33`** — понизить до 30 (минимум для `AccessibilityService.takeScreenshot()`).
   Все вызовы API 31+ обернуть в проверки версии. Окна-полосы Live blur
   (`isCrossWindowBlurEnabled`, API 31) на Android 11 недоступны, оставить режим Snapshot.
3. **Угол шарнира.** Проверить, отдаёт ли Duo 1 непрерывный угол:
   `adb shell dumpsys sensorservice | grep -i hinge`.
   - Есть `TYPE_HINGE_ANGLE` (API 30+) — использовать `fold/HingeAngleSource.kt` как есть.
   - Нет — искать вендорский сенсор Surface Duo или упасть на Jetpack WindowManager
     (`FoldingFeature`): будут только позы, плавный угол потеряется
     (аналог "stops-only" режима из README).
4. **Логика экранов** (`fold/Panels.kt`, `overlay/PanelEngine.kt`, `overlay/FoldOverlayService.kt`):
   привязана к "внутренний/внешний экран" и к паузе при переключении панелей.
   У Duo оба экрана горят вместе — переписать определение половин и положение линии
   сгиба (по центру растянутого дисплея, учесть физический зазор шарнира).

## Что работает без изменений
- Скриншот через `AccessibilityService.takeScreenshot()` (API 30+).
- Оверлей поверх окон.
- Блокировки "restricted settings" на Android 11/12 нет — доступность включается проще.

## План
1. ~~Выяснить версию Android и наличие сенсора угла~~ — сделано, см. раздел ниже.
2. Понизить `minSdk` до 30, убрать/обернуть вызовы API 31+ и 33+.
3. Портировать `duo_unfold.agsl` в GLSL ES, заменить `RuntimeShader` в `fold/DuoShader.kt`
   и `overlay/FoldOverlayView.kt` на GL-рендер.
4. Адаптировать `Panels.kt` и `PanelEngine.kt` под растянутый дисплей Duo.
5. Проверить на устройстве: `adb logcat -s DuoOverlay`, `adb shell am broadcast -a com.duoopen.DEMO`.

## Риски
- Нет непрерывного угла шарнира — эффект не будет следовать за рукой.
- Производительность GL-рендера на Snapdragon 855 при частом перезахвате экрана.
- Скриншот растянутого дисплея может отличаться по размеру и содержать зазор шарнира.

## Результаты проверки на устройстве (21.10.2026)
- `ro.build.version.sdk` = **32**.
- `dumpsys sensorservice` показывает два датчика: `Hinge Angle Non-wakeup` (handle 0x16b) и
  `Hinge Angle Wakeup` (handle 0x16c), производитель Microsoft, тип `android.sensor.hinge_angle(36)`,
  `perm: n/a` (разрешения не нужны). Системная оболочка подписана на датчик, значит он рабочий.
- Это стандартный `TYPE_HINGE_ANGLE` (API 30+), `HingeAngleSource.kt` должен подойти почти без изменений.
- Пока неизвестно: непрерывный ли угол или только стопы 0/90/180 (в выводе нет строки resolution).
  Проверить через **Tune -> Copy sensor report** после первой сборки.
- Вывод: блокер по сенсору снят, остаётся порт шейдера (AGSL -> GLSL ES) и понижение `minSdk` до 30.

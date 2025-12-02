## Полная инструкция по созданию Debian-пакета Winicar v5.0.1.1 

### 0. Подготовительные шаги и наименования

*   **Корневой каталог проекта:** `/home/user/Winicar/`
*   **Исполняемый файл:** `Winicar` (будет сгенерирован в папке сборки)
*   **Версия пакета:** `5.0.1.1`
*   **Папка для сборки релиза:** `build_release/`
*   **Папка для упаковки:** `deploy/`

#### 0.1. Установка инструментов

```bash
sudo apt update
sudo apt install -y build-essential cmake git debhelper devscripts libfuse2
```

#### 0.2. Переменные окружения Qt

Предполагаем, что Qt 6.8.3 установлен в `/opt/Qt/6.8.3/gcc_64/`. Установите переменные для текущей сессии:

```bash
export QT_INSTALL_DIR=/opt/Qt/6.8.3/gcc_64
export PATH=$QT_INSTALL_DIR/bin:$PATH
# Проверка
qmake6 -version
```

### 1. Этап: Сборка Релиза (Winicar и LimeReport)

Перейдите в корневой каталог:

```bash
cd /home/user/Winicar/
```

#### 1.1. Сборка (если не собрано)

Создайте папку для релизной сборки и сконфигурируйте CMake.

```bash
mkdir build_release
cd build_release

# Конфигурация CMake
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_PREFIX_PATH=$QT_INSTALL_DIR \
    # Добавьте любые другие важные флаги для вашего проекта, 
    # если они есть, например, путь к LimeReport если он не в корне

# Сборка проекта
cmake --build . --parallel $(nproc)
```
> **Результат:** Исполняемый файл `Winicar` должен появиться в `build_release/`.

### 2. Этап: Приватное Развертывание (Сбор зависимостей)

Мы используем `linuxdeployqt` только как "сборщик зависимостей", чтобы собрать все необходимые библиотеки Qt.

#### 2.1. Подготовка папки развертывания

Вернитесь в корень и создайте структуру пакета:

```bash
cd /home/user/Winicar/

# Создаем папку, которая будет скопирована в пакет. 
# Приложение будет установлено в /opt/winicar-v5
mkdir -p deploy/opt/winicar-v5 

# Копируем релизный исполняемый файл
cp build_release/Winicar deploy/opt/winicar-v5/ 

# Копируем ресурсы, которые должны быть рядом с исполняемым файлом (если не в QRC)
# Например, иконки, внешние конфиги и т.д.
cp -r Resources/* deploy/opt/winicar-v5/ # Если в этой папке что-то нужно
```

#### 2.2. Включение LimeReport (Критично!)

Если LimeReport был собран как **динамическая библиотека** (`.so`), её нужно скопировать рядом.

```bash
# ПРИМЕР: Если LimeReport собрана в build_release/LimeReport/libLimeReport.so
cp build_release/LimeReport/libLimeReport.so deploy/opt/winicar-v5/ 
# ИЛИ: создайте подпапку lib, если хотите имитировать стандартную структуру
# mkdir -p deploy/opt/winicar-v5/lib
# cp build_release/LimeReport/libLimeReport.so deploy/opt/winicar-v5/lib/
```

#### 2.3. Сбор зависимостей Qt с помощью `linuxdeployqt`

Мы используем его, чтобы скопировать Qt-библиотеки и плагины.

```bash
# 1. Скачиваем linuxdeployqt (если не скачан)
wget "https://github.com/probonopd/linuxdeployqt/releases/download/continuous/linuxdeployqt-continuous-x86_64.AppImage" -O linuxdeployqt
chmod +x linuxdeployqt

# 2. Переходим в каталог, куда будут скопированы все библиотеки Qt (ВАЖНО!)
cd deploy/opt/winicar-v5

# 3. Запуск linuxdeployqt (только для сбора, без флага -appimage)
# Он найдет qmake6 в PATH и возьмет оттуда библиотеки 6.8.3
../../linuxdeployqt Winicar -bundle-non-qt-libs
```
> **Что произошло:** Внутри `deploy/opt/winicar-v5/` появились папки `lib/`, `plugins/`, `translations/` и т.д., содержащие приватные копии Qt 6.8.3.

### 3. Этап: Создание Debian-структуры (`debian/`)

Вернитесь в корень проекта (`/home/user/Winicar/`).

#### 3.1. Создание папки `debian/`

```bash
cd /home/user/Winicar/
mkdir debian
```

#### 3.2. `deploy/opt/winicar-v5/launcher.sh` (Скрипт запуска - **Критично!**)

Создайте этот файл и сделайте его исполняемым, чтобы наше приложение нашло приватные библиотеки Qt.

```bash
# Содержимое deploy/opt/winicar-v5/launcher.sh
#!/bin/bash
# Скрипт-обертка для запуска Winicar v5.0.1.1

# Определяем путь установки (/opt/winicar-v5)
APP_DIR=$(dirname "$0")

# Указываем, где искать приватные библиотеки Qt и плагины
# LD_LIBRARY_PATH указывает путь к .so файлам (Core, GUI, Widgets, LimeReport)
export LD_LIBRARY_PATH=$APP_DIR/lib:$LD_LIBRARY_PATH
# QT_QPA_PLATFORM_PLUGIN_PATH указывает, где лежит libqxcb.so
export QT_QPA_PLATFORM_PLUGIN_PATH=$APP_DIR/plugins/platforms

# Запускаем приложение
exec "$APP_DIR/Winicar" "$@"
```
> **Сделать исполняемым:**
> `chmod +x deploy/opt/winicar-v5/launcher.sh`

#### 3.3. `debian/control` (Метаданные)

Укажите только системные зависимости.

```ini
Source: winicar-v5 
Section: utils
Priority: optional
Maintainer: Your Name <your.email@example.com>
Build-Depends: 
    debhelper-compat (= 13),
    cmake
Standards-Version: 4.6.0

Package: winicar-v5 
Version: 5.0.1.1
Architecture: amd64
Depends: 
    ${shlibs:Depends}, 
    ${misc:Depends},
    # Системные зависимости, которые Qt не включает:
    libgl1, 
    libfontconfig1 
    # И другие общие библиотеки, если требуются (проверить на чистой ОС)
Description: Winicar v5.0.1.1. 
 Приложение для обработки ЭКГ с приватными библиотеками Qt 6.8.3.
```

#### 3.4. `debian/install` (Установка файлов)

Указываем, что копируем всю папку `deploy/` в корень пакета (`/`).

```
# Формат: <путь к файлу/папке в deploy> <путь назначения в пакете>

# Копируем всю папку /opt/winicar-v5 с приложением и библиотеками
deploy/opt/winicar-v5 /opt/

# Копируем ярлык для меню приложений
debian/winicar-v5.desktop /usr/share/applications/

# Копируем иконку для отображения
deploy/usr/share/pixmaps/Winicar.png /usr/share/pixmaps/
```
> **Примечание:** Папка `deploy/usr/share/pixmaps/` еще не существует, ее нужно создать (см. 3.7).

#### 3.5. `debian/winicar-v5.desktop` (Ярлык)

```ini
[Desktop Entry]
Type=Application
Name=Winicar v5.0.1.1
Comment=Программа для обработки ЭКГ
# Exec указывает на наш скрипт-обертку!
Exec=/opt/winicar-v5/launcher.sh
Icon=Winicar # Имя файла иконки без расширения
Categories=Utility;Application;
Terminal=false
```

#### 3.6. `debian/rules` (Скрипт сборки)

```make
#!/usr/bin/make -f
# Так как мы собрали и развернули вручную, просто используем debhelper

%:
    dh $@
```

#### 3.7. Остальные обязательные файлы

```bash
# Создайте иконку (например, 256x256) и скопируйте ее
# Вам нужно взять иконку из проекта, например, из icon.rc или создать из .png
# Для примера:
cp /путь/к/вашей/иконке/Winicar.png deploy/usr/share/pixmaps/

# Создание остальных обязательных файлов
dch --create -v 5.0.1.1 "Initial release of Winicar v5.0.1.1"
cp /path/to/your/LICENSE debian/copyright
```

### 4. Этап: Финальная Сборка `.deb`

Убедитесь, что вы находитесь в корневом каталоге `/home/user/Winicar/`.

```bash
# Очистка
dpkg-buildpackage --clean

# Сборка бинарного пакета без подписи
dpkg-buildpackage -b --no-sign
```

### 5. Этап: Распространение и Установка

1.  Готовый файл **`winicar-v5_5.0.1.1_amd64.deb`** будет на один каталог выше.
2.  Перенесите его на флешке на машину с Astra Linux.
3.  На Astra Linux в терминале:
    ```bash
    # Установка пакета
    sudo dpkg -i winicar-v5_5.0.1.1_amd64.deb
    
    # Решение возможных проблем с зависимостями системных библиотек (libgl1, libfontconfig1 и т.п.)
    sudo apt install -f
    ```

**Результат:** Приложение `Winicar v5.0.1.1` установлено в `/opt/`, иконка и ярлык появились в меню Astra Linux, и оно использует свои приватные библиотеки Qt 6.8.3, независимо от того, что есть в системе.

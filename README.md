# llm-stack-opensuse
Локальный LLM-стек на openSUSE: Ollama + Open WebUI (CPU-режим)

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![Platform](https://img.shields.io/badge/Platform-openSUSE%20Leap%2016.0-73BA25?logo=opensuse&logoColor=white)](https://www.opensuse.org/)
[![Ollama](https://img.shields.io/badge/Ollama-000000?logo=ollama&logoColor=white)](https://ollama.com/)
[![Open WebUI](https://img.shields.io/badge/Open%20WebUI-00B4D8?logo=openwebui&logoColor=white)](https://github.com/open-webui/open-webui)

**Автор:** LeO (Hallow)
**Дата:** октябрь 2026
**Лицензия:** CC BY-SA 4.0

---

## О чём этот документ

Полное, воспроизводимое руководство по развёртыванию локального стека языковых моделей на openSUSE Leap 16.0 **без использования GPU**. Конфигурация проверена в реальной эксплуатации: Ollama как сервер моделей, Open WebUI как веб-интерфейс, доступ по локальной сети и через временный внешний туннель.

Документ написан после того, как я **дважды** потерял рабочую конфигурацию при переустановке системы. Цель - чтобы восстановление занимало 30-40 минут, а не полный рабочий день проб и ошибок.

---

## Содержание

- [Архитектура](#архитектура)
- [Почему CPU, а не GPU](#почему-cpu-а-не-gpu)
- [Требования](#требования)
- [Этап 1. Установка Ollama](#этап-1-установка-ollama)
- [Этап 2. Настройка доступа по сети](#этап-2-настройка-доступа-по-сети)
- [Этап 3. Установка Open WebUI в Docker](#этап-3-установка-open-webui-в-docker)
- [Этап 4. Загрузка моделей](#этап-4-загрузка-моделей)
- [Этап 5. Настройка моделей в Open WebUI](#этап-5-настройка-моделей-в-open-webui)
- [Этап 6. Доступ по локальной сети](#этап-6-доступ-по-локальной-сети)
- [Этап 7. Временный внешний доступ](#этап-7-временный-внешний-доступ)
- [Альтернатива: GPU на неподдерживаемых AMD-картах](#альтернатива-gpu-на-неподдерживаемых-amd-картах)
- [Известные грабли](#известные-грабли)
- [Шпаргалка на каждый день](#шпаргалка-на-каждый-день)
- [Диагностика](#диагностика)
- [Ссылки](#ссылки)

---

## Архитектура

```
+---------------------------------------------------------+
|                openSUSE Leap 16.0                       |
|                                                         |
|   +------------------+        +---------------------+   |
|   |     Ollama       |        |    Open WebUI       |   |
|   |  (systemd, хост) | <----> |  (Docker, :3000)    |   |
|   |  0.0.0.0:11434   |  API   |  :8080 внутри       |   |
|   +------------------+        +---------------------+   |
|            |                            |               |
|            |                            |               |
|   +--------+----------------------------+-----------+   |
|   |  Модели: gemma3:4b, dolphin-llama3:8b           |   |
|   |  Хранение: ~/.ollama/models/                    |   |
|   +-------------------------------------------------+   |
|                                                         |
+---------------------------------------------------------+
             |                              |
             | локальная сеть               | внешний доступ
             v                              v
   http://192.168.x.x:3000      https://xxxx.trycloudflare.com
```

| Компонент | Роль | Порт | Запуск |
|-----------|------|------|--------|
| Ollama | Сервер моделей | 11434 | systemd |
| Open WebUI | Веб-интерфейс | 3000 | Docker |

---

## Почему CPU, а не GPU

На моей машине установлена **AMD Radeon RX 5700 XT** (архитектура **gfx1010**, RDNA1, Navi 10). Официальная сборка Ollama **не видит эту карту** и молча переключается на CPU.

Корень проблемы - не в отсутствии поддержки на уровне ROCm (система карту видит), а в том, что в коде Ollama список `supported_types` для amdgpu оказывается пустым. Это задокументировано в issue [#8806](https://github.com/ollama/ollama/issues/8806).

Проверить, что происходит в системе:

```bash
ollama ps
```
Если в колонке ```PROCESSOR``` стоит ```100% CPU``` - GPU не задействован, даже если установщик написал AMD GPU ready.

Эта инструкция сознательно описывает CPU-путь. Он предсказуем, стабилен и не ломается при обновлениях. Вариант с GPU для неподдерживаемых карт вынесен в отдельный раздел с честным описанием рисков.

## Требования
- openSUSE Leap 16.0 (или Tumbleweed)

- Пользователь с ```sudo```

- Docker и Docker Compose

- Интернет

- ОЗУ: минимум 8 ГБ, комфортно - 16 ГБ

- Диск: минимум 20 ГБ под модели

Проверка готовности:

```bash
docker --version
docker ps
```
## Этап 1. Установка Ollama
### 1.1. Установить zstd
Скрипт установки Ollama распаковывает ```.tar.zst,``` для этого нужен ```zstd```:

```bash
sudo zypper install -y zstd
```
### 1.2. Установить Ollama официальным скриптом
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Скрипт:

- скачивает свежий бинарник в /usr/local/bin/ollama

- создаёт системного пользователя ollama

- создаёт и запускает systemd-сервис

### 1.3. Исправить путь к команде
Если в системе раньше стоял пакет ```ollama``` из репозитория, в ```/usr/bin``` может остаться «мёртвый» путь. Создаём симлинк:

```bash
sudo ln -sf /usr/local/bin/ollama /usr/bin/ollama
```
Проверка:

```bash
ollama --version
```
### 1.4. Проверить, что сервис работает
```bash
sudo systemctl status ollama
curl http://localhost:11434
```
Ожидаемый ответ: Ollama is running

## Этап 2. Настройка доступа по сети
По умолчанию Ollama слушает только ```127.0.0.1```. Контейнер Open WebUI находится в изолированной сети и до ```127.0.0.1``` хоста не достучится. Нужно разрешить прослушивание всех интерфейсов.

### 2.1. Создать override для systemd
```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```
Содержимое файла:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```
Сохранить: ```Ctrl+O```, ```Enter```. Выйти: ```Ctrl+X```.

### 2.2. Применить
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```
### 2.3. Проверить
```bash
sudo ss -tlnp | grep 11434
```
Должно быть ```*:11434``` или ```0.0.0.0:11434.```


Узнать свой IP:

```bash
ip a | grep -E "inet " | grep -v 127.0.0.1
```
Например: ```192.168.1.109```

Проверка по IP:

```bash
curl http://192.168.1.109:11434
```
Запомните свой IP. Он понадобится на этапе 3 и при каждой перенастройке контейнера.

## Этап 3. Установка Open WebUI в Docker
### 3.1. Проверить, что порт 3000 свободен
```bash
sudo ss -tlnp | grep 3000
```
Если занят - выбрать другой порт (например, 3002) и заменить 3000 во всех командах ниже.

### 3.2. Запустить контейнер
Замените ```192.168.1.109``` на ваш IP из этапа 2.3:

```bash
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://192.168.1.109:11434 \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```
Ключевые параметры:

| Параметр | Зачем |
|----------|-------|
| `-p 3000:8080` | Наружу порт 3000, внутри контейнера 8080 |
| `-v open-webui:/app/backend/data` | Том: аккаунты, чаты, настройки |
| `-e OLLAMA_BASE_URL=...` | **Прямой IP хоста**, не `host.docker.internal` |
| `--restart always` | Автозапуск после перезагрузки |

### 3.3. Проверить
```bash
sleep 5
docker ps | grep open-webui
docker exec open-webui curl -s http://192.168.1.109:11434
```
Ожидаемый ответ: ```Ollama is running```

### 3.4. Открыть в браузере
```text
http://localhost:3000
```
Создать аккаунт. Первый пользователь автоматически становится администратором.

## Этап 4. Загрузка моделей
### 4.1. Рабочие модели (проверены)
```bash
# Универсальная, стабильная, хорошо понимает русский
ollama pull gemma3:4b

# Без цензуры, стабильна
ollama pull dolphin-llama3:8b
```
### 4.2. Модели с поддержкой инструментов (для тестов)
Для Open WebUI, если нужен tool calling:

```bash
# 128K контекст, 3B параметров, поддержка tools
ollama pull granite4:3b

# Классика для tool calling
ollama pull llama3.1:8b
```
### 4.3. Проверка
```bash
ollama list
```
### 4.4. Что НЕ ставить

| Модель | Почему |
|--------|--------|
| `qwen3:8b` | Контекст 4096 по умолчанию, thinking-режим, на CPU крайне медленно |
| `gemma3:12b` | В Q4 занимает ~7-8 ГБ, на CPU будет очень медленно |

## Этап 5. Настройка моделей в Open WebUI
### 5.1. Отключить «Встроенные инструменты»
```gemma3:4b``` и ```dolphin-llama3:8b``` не поддерживают нативный вызов инструментов. Если оставить эту опцию включённой, Open WebUI (начиная с версии 0.10.1) будет пытаться передать модели tools, получит ```does not support tools```, и чат зависнет.

### Путь через чат:

Открыть чат с моделью

- Иконка шестерёнки в панели управления (или «плюс» вверху справа)

- Найти раздел «Встроенные инструменты» (Builtin Tools)

- Снять галочку

- Сохранить

### Путь через Рабочее пространство (если через чат не находится):

- Иконка пользователя внизу слева → Рабочее пространство (Workspace)

- Модели (Models)

- Если список пуст - это нормально. Нажать «Создать модель»

- Базовая модель: ```gemma3:4b```

- Дать название, сохранить

- Открыть созданную модель → снять «Встроенные инструменты»

Почему «Рабочее пространство» пустое: это конструктор кастомных «обёрток», а не список скачанных моделей. Скачанные модели видны в основном чате, в Workspace их нет, пока не создадите на их основе кастомную.

### 5.2. Если модели не видны в интерфейсе
Админ-панель → Настройки → Подключения → Управление соединениями API OpenAI:

- Base URL: ```http://192.168.1.109:11434/v1```

- API Key: ```ollama``` (любая строка)

Сохранить, нажать «Обновить» рядом с соединением.

## Этап 6. Доступ по локальной сети
### 6.1. Firewall
На openSUSE Leap 16.0 ```firewall-cmd``` падает с ошибкой ```ImportError: pyexpat... undefined symbol: XML_SetAllocTrackerActivationThreshold```. Это сломанный Python 3.13 в системе.

Рабочее решение - отключить firewalld:

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```
Если firewalld у вас исправен:

```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```
### 6.2. Проверка
С телефона в той же Wi-Fi сети:

``` text
http://192.168.1.109:3000
```
## Этап 7. Временный внешний доступ
### 7.1. Скачать cloudflared
```bash
cd ~
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64
chmod +x cloudflared-linux-amd64
```
Не перемещать в ```/usr/local/bin```. Запускать напрямую из домашней папки.

### 7.2. Запуск в foreground
```bash
./cloudflared-linux-amd64 tunnel --url http://localhost:3000
```
В логе появится ссылка:

```text
https://xxxx-xxxx-xxxx.trycloudflare.com
```
Отправить её тому, кому нужен доступ. Работает, пока терминал открыт.

### 7.3. Запуск в фоне
```bash
cd ~
nohup ./cloudflared-linux-amd64 tunnel --url http://localhost:3000 > ~/tunnel.log 2>&1 &
sleep 10
grep -o 'https://[^ ]*trycloudflare.com' ~/tunnel.log | head -1
```
### 7.4. Остановить
```bash
pkill -f cloudflared
```
### 7.5. Предупреждение
Ссылка без пароля. Любой, кто её знает, получает доступ к вашему AI. Для кратковременной демонстрации - приемлемо. Для постоянного использования - нужен named tunnel с аутентификацией.

### Альтернатива: GPU на неподдерживаемых AMD-картах
### Что известно
- Официальная Ollama не видит gfx1010 из-за пустого списка supported_types (issue #8806).

- Форк ```likelovewant/ollama-for-amd``` явно включает ```gfx1010:xnack-``` в список поддерживаемых.

В обсуждениях форка есть успешные кейсы с Llama 3.2 на RX 5700 XT.

### Что потребуется
1. Удалить официальную Ollama.

2. Установить кастомную сборку ```lollama-for-amd```l под подходящую версию ROCm (по сообщениям - ROCm 5.7 стабильнее для gfx1010, чем 6.x).

3. Настроить переменную окружения:

```bash
sudo systemctl edit ollama
```
```ini
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=10.3.0"
```
4. Проверить, что модель грузится на GPU:

```bash
ollama ps
```
Должно быть ```100% GPU``` или смешанный режим.

### Риски
- Хрупкость: обновление системы может сломать связку ROCm + кастомная сборка.

- Частичная загрузка: при нехватке VRAM модель уходит в shared memory, выигрыш от GPU падает.

- Молчаливые падения: PyTorch может откатиться на CPU без явной ошибки.

- Воспроизводимость: конфигурация, работающая на одной версии ROCm, может не работать на другой.

Рекомендация: экспериментировать на отдельной копии системы или в виртуальной машине, не на рабочей конфигурации.

### Известные грабли

| Симптом | Причина | Решение |
|---------|---------|---------|
| `ollama: command not found` | Бинарник в `/usr/local/bin`, PATH смотрит в `/usr/bin` | `sudo ln -sf /usr/local/bin/ollama /usr/bin/ollama` |
| `ollama ps` показывает `100% CPU` | GPU не поддерживается | CPU-режим, см. [раздел про GPU](#sec-gpu-alt) |
| `firewall-cmd` падает | Сломан `pyexpat` в Python 3.13 | Отключить firewalld |
| Модель висит, ошибка `does not support tools` | Open WebUI 0.10.1+ включает native tool calling | Снять «Встроенные инструменты» |
| Порт 8080 занят | stirling-pdf или другой контейнер | Open WebUI на 3000 |
| `host.docker.internal` не резолвится | Особенность Docker на Linux | Использовать прямой IP хоста |
| Рабочее пространство пустое | Это конструктор, а не список моделей | Создать кастомную модель или настраивать через чат |
| Open WebUI отдаёт `500 Internal Error` | Повреждён том или конфиг | Пересоздать том `open-webui` (потеряются чаты) |

## Шпаргалка на каждый день
### Проверка состояния

```bash
curl http://localhost:11434          # Ollama
docker ps | grep open-webui          # WebUI
ollama list                          # Модели
ollama ps                            # Что загружено сейчас
```
### Управление Ollama

```bash
sudo systemctl start ollama
sudo systemctl stop ollama
sudo systemctl restart ollama
sudo systemctl status ollama
```
### Управление Open WebUI

```bash
docker start open-webui
docker stop open-webui
docker restart open-webui
docker logs open-webui --tail 30
```
### Модели

```bash
ollama pull <модель>       # Скачать
ollama rm <модель>         # Удалить
ollama run <модель>        # Проверить в терминале
```
### Туннель

```bash
# Запуск в фоне
cd ~
nohup ./cloudflared-linux-amd64 tunnel --url http://localhost:3000 > ~/tunnel.log 2>&1 &
sleep 10
grep -o 'https://[^ ]*trycloudflare.com' ~/tunnel.log | head -1

# Остановка
pkill -f cloudflared
```
### Диагностика
### Open WebUI не открывается
```bash
docker ps | grep open-webui
docker logs open-webui --tail 30
sudo ss -tlnp | grep 3000
```
### Модели не видны в интерфейсе

```bash
docker exec open-webui curl -s http://192.168.1.109:11434/api/tags
```
Если пусто - неверный ```OLLAMA_BASE_URL```. Пересоздать контейнер с правильным IP.

### Модель не отвечает / чат висит
Скорее всего, включены «Встроенные инструменты». См. этап 5.1.

### Ollama не отвечает
```bash
sudo systemctl status ollama
journalctl -u ollama -n 50 --no-pager
sudo systemctl restart ollama
```
### IP изменился
DHCP может выдать новый адрес. Тогда:

1. Пересоздать контейнер Open WebUI с новым IP

2. Обновить ссылку для доступа по сети

Чтобы IP не менялся - настроить статическую привязку MAC в роутере.

## Ссылки

- [Ollama](https://github.com/ollama/ollama)
- [Open WebUI](https://github.com/open-webui/open-webui)
- [Cloudflare Tunnel](https://github.com/cloudflare/cloudflared)
- [ollama-for-amd (форк)](https://github.com/likelovewant/ollama-for-amd)
- [Issue #8806: amdgpu not detected](https://github.com/ollama/ollama/issues/8806)
- [model-val: валидация AI-моделей](https://github.com/Iso-Hallow/model-val)

---

<a id="sec-changelog"></a>
## История изменений

| Дата | Изменение |
|------|-----------|
| 2026-06 | Первая сборка стека, эксперименты с ROCm |
| 2026-09 | Потеря конфигурации при переустановке, восстановление |
| 2026-10 | Стабилизация на CPU-режиме, оформление гайда |

---

*Документ распространяется по лицензии [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/). При использовании указывайте авторство: LeO (Hallow).*

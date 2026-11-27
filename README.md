# Сборка двух портов в OPNsense 26.7

Задание: в GitHub Actions с помощью образа [vmactions/opnsense-vm](https://github.com/vmactions/opnsense-vm) собрать в OPNsense 26.7 amd64 два порта из дерева FreeBSD Ports — `security/sudo` и `sysutils/tmux`, сформировать из них собственный репозиторий командой `pkg repo` и приложить архив репозитория к GitHub Release.

Работа ведётся с Linux. Локальная VM OPNsense не требуется: сборка выполняется в `vmactions/opnsense-vm` внутри GitHub Actions, а `pkg repo` и упаковка архива — в FreeBSD-контейнере на Linux.

## 1. Порты и среда

| Origin порта | Пакет | Назначение | Команда проверки |
| --- | --- | --- | --- |
| `security/sudo` | `sudo` | Выполнение команд от имени другого пользователя | `sudo -V` |
| `sysutils/tmux` | `tmux` | Терминальный мультиплексор | `tmux -V` |

Сборка выполняется официальными инструментами OPNsense (`opnsense/tools`), которые сами обновляют дерево портов и складывают пакеты в `/usr/tools/sets/`. Конфигурация рассчитана на OPNsense Community Edition 26.7 amd64 (база FreeBSD 15.1). Фактические версии фиксируются в журнале и в `BUILD-INFO.txt`.

## 2. Файлы проекта

| Путь | Назначение |
| --- | --- |
| `.github/workflows/build-sudo.yml` | Ручной запуск VM, сборка `security/sudo`, выгрузка пакета в артефакт |
| `.github/workflows/build-tmux.yml` | Ручной запуск VM, сборка `sysutils/tmux`, выгрузка пакета в артефакт |
| `.gitattributes` | Окончания строк LF для YAML и скриптов |
| `.gitignore` | Исключение временных каталогов и артефактов из Git |

Оба workflow запускаются вручную (`workflow_dispatch`) и независимо друг от друга: каждый поднимает свою VM OPNsense, клонирует `opnsense/tools`, обновляет дерево портов и собирает один пакет. Результат каждого прогона — артефакт `opnsense-sudo-package` или `opnsense-tmux-package` со содержимым `/usr/tools/sets/`.

## 3. Пробный запуск

На вкладке **Actions**:

1. **Build - sudo** → **Run workflow**, ветка `main`.
2. **Build - tmux** → **Run workflow**, ветка `main`.

| Этап | Что искать в журнале |
| --- | --- |
| Checkout | `actions/checkout@v4` клонирует репозиторий на runner |
| Запуск VM | `OPNsense: 26.7`, `FreeBSD 15.1-... amd64` |
| Подготовка | `pkg install -y git` внутри VM |
| Клонирование tools | `git clone https://github.com/opnsense/tools` в `/usr/tools` |
| Обновление Ports | `make update` в `/usr/tools`, SHA коммита дерева Ports |
| Сборка | `make ports PORTSLIST="security/sudo"` (или `sysutils/tmux`), фактическое время |
| Выгрузка | Артефакт `opnsense-sudo-package` / `opnsense-tmux-package` в **Summary** |

После успешных прогонов скачать оба артефакта со страницы прогонов. В каждом будет каталог `sets/` с собранным `.pkg`.

## 4. Сборка репозитория и публикация GitHub Release

Так как отдельного workflow для `pkg repo` и релиза нет, эти шаги выполняются на Linux из скачанных артефактов. `pkg repo` запускается в FreeBSD-контейнере — это не OPNsense, а обычный FreeBSD 15.1, совместимый по ABI.

### 4.1. Подготовка каталога

```bash
# Артефакты скачаны и распакованы в каталоги sudo/ и tmux/
mkdir -p lab/repo/All
cp sudo/sets/*.pkg  lab/repo/All/
cp tmux/sets/*.pkg  lab/repo/All/

# BUILD-INFO.txt
{
  echo "OPNsense: 26.7"
  echo "FreeBSD: 15.1"
  echo "Arch: amd64"
  echo "Ports: security/sudo, sysutils/tmux"
  echo "Built: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
} > lab/BUILD-INFO.txt

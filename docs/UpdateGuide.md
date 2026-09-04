# Подробный учебник: безопасное обновление BikeShop на ThinkPad

> Runbook — это пошаговая инструкция для повторяемой операции. В данном случае: как заменить старую опубликованную версию BikeShop новой, не потеряв базу, секреты, изображения и Data Protection keys.
>
> Для обычного обновления используй короткий [UpdateRunbook.md](UpdateRunbook.md). Этот файл объясняет, почему каждый шаг нужен, и содержит более развёрнутый rollback.

## Когда нужен этот документ

Используй его после изменений в BikeShop, когда новая версия уже:

- закоммичена в репозиторий;
- успешно собирается и запускается на Desktop;
- опубликована для `linux-x64`.

Не начинай обновление, если локальная версия не проверена. Сервер — не место для первой проверки компиляции.

## Главное правило

На сервере есть два вида файлов:

| Можно заменить publish-версией | Нельзя случайно удалить |
|---|---|
| DLL, зависимости, конфигурация без секретов, статические файлы из новой публикации | SQL Docker volume, `/etc/bikeshop/*.env`, `.aspnet/DataProtection-Keys`, пользовательские изображения, если они появились после публикации |

Поэтому **не выполняй** `sudo rm -rf /opt/bikeshop/api` или `sudo rm -rf /opt/bikeshop/blazor` как часть обычного обновления. Так можно удалить Data Protection keys, а в случае Blazor — и загруженные через admin UI изображения.

## Перед началом

На Desktop, в корне репозитория BikeShop:

```bash
git status
dotnet build BikeShop.sln -c Release
```

`git status` должен быть понятным: ты должен знать, какие изменения попадут в коммит. `dotnet build` должен завершиться без ошибок.

Затем проверь новую версию приложения локально так, как обычно делаешь во время разработки.

## Шаг 1. Опубликовать новую версию на Desktop

Из корня репозитория выполни:

```bash
dotnet publish BikeShop.API/BikeShop.API.csproj \
  -c Release -r linux-x64 --self-contained false \
  -o ./publish/api

dotnet publish BikeShop.Blazor/BikeShop.Blazor.csproj \
  -c Release -r linux-x64 --self-contained false \
  -o ./publish/blazor
```

Перед переносом убедись, что внутри publish-папок действительно есть:

```text
publish/api/BikeShop.API.dll
publish/blazor/BikeShop.Blazor.dll
```

## Шаг 2. Сделать минимальную точку восстановления

На ThinkPad проверь, что текущая версия работает:

```bash
systemctl status bikeshop-api bikeshop-blazor nginx --no-pager
sudo docker ps
```

Затем зафиксируй, какая версия была установлена. Самый простой вариант — записать SHA коммита в файл рядом с publish:

```bash
git rev-parse --short HEAD
```

Запиши результат в заметки обновления. Если новая версия окажется плохой, будет ясно, к какому коммиту возвращаться.

> Полноценная стратегия backup базы — отдельная тема. В этом lab мы хотя бы не трогаем Docker volume и не выполняем опасные команды удаления.

## Шаг 3. Перенести publish на сервер во временную папку

Новая версия сначала должна попасть **во временный каталог**, а не поверх работающего приложения. Например:

```bash
mkdir -p /tmp/bikeshop-update/api
mkdir -p /tmp/bikeshop-update/blazor
```

Перенеси туда содержимое `publish/api` и `publish/blazor` с Desktop любым уже настроенным способом: `scp`, `rsync` или копированием через SSH/SFTP. Важно сохранить структуру файлов внутри каждой publish-папки.

После передачи на ThinkPad проверь наличие главных DLL:

```bash
ls -l /tmp/bikeshop-update/api/BikeShop.API.dll
ls -l /tmp/bikeshop-update/blazor/BikeShop.Blazor.dll
```

Если файла нет — остановись. Не перезапускай службы и не заменяй текущую версию.

## Шаг 4. Остановить только приложения

Не останавливай Nginx и SQL Server: они не требуют обновления приложения.

```bash
sudo systemctl stop bikeshop-api
sudo systemctl stop bikeshop-blazor
```

В этот момент сайт временно недоступен или показывает ошибку прокси. База продолжает жить, данные не удаляются.

## Шаг 5. Заменить файлы, сохранив состояние

Сначала сохраним важные каталоги в безопасном временном месте:

```bash
sudo mkdir -p /tmp/bikeshop-state/api /tmp/bikeshop-state/blazor
sudo cp -a /opt/bikeshop/api/.aspnet /tmp/bikeshop-state/api/
sudo cp -a /opt/bikeshop/blazor/.aspnet /tmp/bikeshop-state/blazor/
```

Если админка уже сохраняет новые изображения на сервер, сохрани и их каталог. Его точный путь зависит от реализации upload; сначала посмотри, где в `/opt/bikeshop/blazor/wwwroot/images` лежат добавленные после публикации файлы.

Затем скопируй новую публикацию поверх существующей папки. Используй `rsync`, потому что он заменяет файлы аккуратно и не требует удалять корневую папку:

```bash
sudo rsync -a /tmp/bikeshop-update/api/ /opt/bikeshop/api/
sudo rsync -a /tmp/bikeshop-update/blazor/ /opt/bikeshop/blazor/
```

Верни сохранённые Data Protection keys:

```bash
sudo cp -a /tmp/bikeshop-state/api/.aspnet /opt/bikeshop/api/
sudo cp -a /tmp/bikeshop-state/blazor/.aspnet /opt/bikeshop/blazor/
```

И снова назначь владельца файлов приложению:

```bash
sudo chown -R bikeshop:bikeshop /opt/bikeshop/api
sudo chown -R bikeshop:bikeshop /opt/bikeshop/blazor
```

### Почему здесь нет `rsync --delete`

`--delete` удаляет на сервере файлы, которых нет в новой publish-папке. Это удобно для чистой синхронизации, но опасно здесь: publish не содержит `.aspnet/DataProtection-Keys` и может не содержать загруженные изображения. Без отдельной хорошо продуманной схемы данных мы не используем `--delete`.

## Шаг 6. Запустить новую версию

```bash
sudo systemctl start bikeshop-api
sudo systemctl start bikeshop-blazor
```

Подожди несколько секунд и проверь:

```bash
systemctl status bikeshop-api bikeshop-blazor --no-pager
sudo journalctl -u bikeshop-api -n 80 --no-pager
sudo journalctl -u bikeshop-blazor -n 80 --no-pager
```

У API должна быть строка вида:

```text
Now listening on: http://127.0.0.1:5001
```

У обеих служб статус должен быть `active (running)`.

## Шаг 7. Проверить сайт с Desktop

Открой:

```text
http://192.168.1.102
```

Минимальный smoke test после каждого обновления:

1. Открывается главная страница и каталог.
2. Открывается карточка товара.
3. Добавление в корзину работает.
4. Вход пользователя работает.
5. Для изменения API: проверить именно сценарий, который изменялся.
6. Для изменения админки: войти как admin и открыть нужную страницу.

Если тесты прошли — обновление завершено.

## Если новая версия не стартовала

### 1. Не паникуй и сначала прочитай журнал

```bash
sudo journalctl -u bikeshop-api -n 150 --no-pager
sudo journalctl -u bikeshop-blazor -n 150 --no-pager
```

Частые причины:

- publish для неверной платформы;
- на сервере нет нужного .NET Runtime;
- ошибка в `appsettings` или `.env`;
- миграция базы не прошла;
- новый код содержит ошибку запуска.

### 2. Вернуться к предыдущей версии

Надёжный rollback возможен, если **до обновления** старая publish-папка была сохранена в отдельный архив/каталог. Поэтому для важных обновлений сначала сделай копию:

```bash
sudo mkdir -p /opt/bikeshop/releases
sudo cp -a /opt/bikeshop/api "/opt/bikeshop/releases/api-before-update"
sudo cp -a /opt/bikeshop/blazor "/opt/bikeshop/releases/blazor-before-update"
```

При rollback:

1. останови API и Blazor;
2. верни сохранённые папки поверх текущих;
3. проверь владельца `bikeshop:bikeshop`;
4. снова запусти службы;
5. проверь сайт и журналы.

> Если новая версия уже изменила схему базы миграцией, откат DLL назад не всегда безопасен. Для текущего учебного проекта перед обновлениями с миграциями особенно важно заранее сделать резервную копию БД. Не пытайся «откатывать миграцию наугад» на единственной копии данных.

## Чего не нужно делать при обычном обновлении

- Не менять Nginx, если маршруты и порты не менялись.
- Не пересоздавать SQL Server container.
- Не удалять Docker volume `bikeshop-sql-data`.
- Не переносить секреты в Git или `appsettings.json`.
- Не запускать приложение вручную в SSH-сессии вместо systemd.
- Не удалять `/opt/bikeshop/api` и `/opt/bikeshop/blazor` целиком.

## Очистка после успешного обновления

Когда новая версия проверена, можно удалить только временные файлы передачи:

```bash
sudo rm -rf /tmp/bikeshop-update /tmp/bikeshop-state
```

Не выполняй эту команду, пока не убедился, что сайт работает: временные каталоги содержат удобную копию новых publish-файлов и сохранённых ключей.

## Итоговый чек-лист

- [ ] Новая версия собрана и проверена на Desktop.
- [ ] Publish сделан для `linux-x64`.
- [ ] На сервер переданы обе publish-папки.
- [ ] Проверено наличие обеих DLL во временных каталогах.
- [ ] Остановлены только API и Blazor.
- [ ] Сохранены `.aspnet/DataProtection-Keys`.
- [ ] Новые файлы скопированы без удаления корневых каталогов.
- [ ] Восстановлены keys и назначен владелец `bikeshop:bikeshop`.
- [ ] API и Blazor имеют статус `active (running)`.
- [ ] Smoke test с Desktop пройден.

# Troubleshooting: BikeShop на ThinkPad

Этот файл нужен, когда BikeShop не работает. Начинай с простых проверок и не перезапускай или не удаляй всё подряд.

## Первая проверка

На ThinkPad выполни:

```bash
systemctl status bikeshop-api bikeshop-blazor nginx --no-pager
sudo docker ps
```

Норма: три службы имеют статус `active (running)`, а контейнер `bikeshop-sql` — `Up`.

## Сайт не открывается с Desktop

1. Открой именно `http://192.168.1.102`, не `https://`.
2. Убедись, что Desktop и ThinkPad подключены к одной домашней сети.
3. На ThinkPad проверь Nginx:

   ```bash
   systemctl status nginx --no-pager
   sudo nginx -t
   ```

4. Если Nginx не работает, прочитай последние сообщения:

   ```bash
   sudo journalctl -u nginx -n 100 --no-pager
   sudo tail -n 100 /var/log/nginx/error.log
   ```

## Браузер открывает сайт, но каталог или корзина показывают ошибку

Вероятнее всего, Blazor работает, но не может получить данные от API.

```bash
systemctl status bikeshop-api --no-pager
sudo journalctl -u bikeshop-api -n 100 --no-pager
```

Проверь API напрямую **с ThinkPad**:

```bash
curl -i http://127.0.0.1:5001/api/products
```

Ожидается HTTP-ответ от API, обычно `200 OK`. Если API не отвечает, не ищи проблему в Nginx: сначала восстанови API или SQL Server.

## API не запускается

```bash
systemctl status bikeshop-api --no-pager
sudo journalctl -u bikeshop-api -n 150 --no-pager
sudo docker ps
```

Частые причины:

- контейнер `bikeshop-sql` не запущен;
- неверное значение в `/etc/bikeshop/api.env`;
- отсутствует нужный ASP.NET Core Runtime;
- новая версия приложения не запускается из-за ошибки конфигурации или миграции.

Проверка установленных runtime:

```bash
dotnet --list-runtimes
```

## Blazor не запускается или показывает ошибку

```bash
systemctl status bikeshop-blazor --no-pager
sudo journalctl -u bikeshop-blazor -n 150 --no-pager
```

Проверь, что API поднят:

```bash
systemctl status bikeshop-api --no-pager
```

Blazor зависит от API; если API остановлен, Blazor не может нормально выполнять запросы к каталогу, корзине и заказам.

## Страница загрузилась, но Blazor не реагирует на клики

Blazor Server использует постоянное SignalR/WebSocket-соединение через `/_blazor`.

1. Обнови страницу браузера один раз.
2. Проверь Blazor и Nginx:

   ```bash
   systemctl status bikeshop-blazor nginx --no-pager
   ```

3. Проверь конфигурацию и перезагрузи Nginx только после успешной проверки:

   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

Не удаляй блок `location /_blazor` и WebSocket-заголовки из Nginx-конфига.

## После изменения unit-файла служба игнорирует изменения

После правки `/etc/systemd/system/bikeshop-*.service` выполни:

```bash
sudo systemctl daemon-reload
sudo systemctl restart bikeshop-api bikeshop-blazor
```

Затем снова проверь статус и журнал.

## После изменения Nginx-конфига сайт перестал открываться

Сначала проверь синтаксис:

```bash
sudo nginx -t
```

Если проверка успешна:

```bash
sudo systemctl reload nginx
```

Если проверка неуспешна, не делай reload. Исправь ошибку в `/etc/nginx/sites-available/bikeshop` и повтори `nginx -t`.

## После обновления приложения появились проблемы

1. Прочитай журналы API и Blazor.
2. Проверь, что версии обеих DLL действительно переданы на сервер.
3. Проверь владельца publish-папок:

   ```bash
   sudo ls -ld /opt/bikeshop/api /opt/bikeshop/blazor
   ```

   Владельцем должен быть `bikeshop`.

4. Убедись, что не потеряны `.aspnet/DataProtection-Keys`.
5. Используй `UpdateRunbook.md`; подробный rollback описан в `UpdateGuide.md`.

## Последняя безопасная мера

Если конфигурация не менялась, а служба временно зависла, её можно перезапустить:

```bash
sudo systemctl restart bikeshop-api
sudo systemctl restart bikeshop-blazor
```

После перезапуска Blazor браузеру может понадобиться обновить страницу: текущее SignalR-соединение будет разорвано и создано заново.

Не удаляй контейнер, Docker volume, `/opt/bikeshop` или `.env` «для починки». Эти действия могут уничтожить данные или секреты и не являются нормальной диагностикой.

Подробная схема и объяснение причин — в `DeploymentGuide.md`.

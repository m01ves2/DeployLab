# BikeShop: краткий runbook обновления

Подробный разбор, причины действий и сценарий rollback — в [UpdateGuide.md](UpdateGuide.md). Здесь только рабочая последовательность.

## 1. На Desktop: проверить и опубликовать

```bash
git status
dotnet build BikeShop.sln -c Release

dotnet publish BikeShop.API/BikeShop.API.csproj \
  -c Release -r linux-x64 --self-contained false \
  -o ./publish/api

dotnet publish BikeShop.Blazor/BikeShop.Blazor.csproj \
  -c Release -r linux-x64 --self-contained false \
  -o ./publish/blazor
```

Проверь новую версию локально до переноса на сервер.

## 2. На ThinkPad: передать новую публикацию во временную папку

Передай содержимое `publish/api` и `publish/blazor` в:

```text
/tmp/bikeshop-update/api
/tmp/bikeshop-update/blazor
```

Убедись, что там есть `BikeShop.API.dll` и `BikeShop.Blazor.dll`.

## 3. Остановить приложения и сохранить keys

```bash
sudo systemctl stop bikeshop-api bikeshop-blazor

sudo mkdir -p /tmp/bikeshop-state/api /tmp/bikeshop-state/blazor
sudo cp -a /opt/bikeshop/api/.aspnet /tmp/bikeshop-state/api/
sudo cp -a /opt/bikeshop/blazor/.aspnet /tmp/bikeshop-state/blazor/
```

## 4. Скопировать новую версию и вернуть keys

```bash
sudo rsync -a /tmp/bikeshop-update/api/ /opt/bikeshop/api/
sudo rsync -a /tmp/bikeshop-update/blazor/ /opt/bikeshop/blazor/

sudo cp -a /tmp/bikeshop-state/api/.aspnet /opt/bikeshop/api/
sudo cp -a /tmp/bikeshop-state/blazor/.aspnet /opt/bikeshop/blazor/

sudo chown -R bikeshop:bikeshop /opt/bikeshop/api /opt/bikeshop/blazor
```

Не используй `rsync --delete` и не удаляй `/opt/bikeshop/*` целиком.

## 5. Запустить и проверить

```bash
sudo systemctl start bikeshop-api bikeshop-blazor
systemctl status bikeshop-api bikeshop-blazor --no-pager
```

На Desktop открой `http://192.168.1.102` и проверь каталог, корзину, вход и изменённый сценарий.

Если запуск не удался, сначала прочитай:

```bash
sudo journalctl -u bikeshop-api -n 150 --no-pager
sudo journalctl -u bikeshop-blazor -n 150 --no-pager
```

# DeployLab

Локальное LAN-развёртывание BikeShop на Ubuntu ThinkPad.

## Документы

| Файл | Роль |
|---|---|
| `README.md` | Карта проекта: что достигнуто, текущий адрес, список документов и scope. |
| `docs\Architecture.md` | Короткая схема: Desktop → Nginx → Blazor/API → SQL Server, порты и расположение компонентов. |
| `docs\Publishing.md` | раткая схема текущего deploy и ключевые команды. build, publish, linux-x64, framework-dependent runtime, локальный smoke test. |
| `docs/DeploymentRunbook.md` | Короткая практическая инструкция: состояние сервера, restart/reboot-проверки, базовые команды служб. |
| `docs/UpdateRunbook.md` | Короткое безопасное обновление уже работающего BikeShop. |
| `docs/Troubleshooting.md` | Короткая диагностика «сайт не открывается / Blazor не видит API / SQL не стартовал / где читать логи». |
| `docs/DeploymentGuide.md` | Подробный учебник: systemd, Docker, Nginx, Data Protection, зависимости и причины решений. |
| `docs/UpdateGuide.md` | Подробный учебник обновления и rollback. |


## Scope

DeployLab покрывает локальное развёртывание BikeShop в домашней сети.
VPS, домен, публичный HTTPS и CI/CD сознательно не входят в этот проект.
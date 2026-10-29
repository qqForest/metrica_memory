---
paths:
  - "**/views*.py"
  - "**/views/**"
---

# Вьюсеты — тонкая view, доступ на запись, разделение

## Тонкая view

View оркестрирует: сериализатор, сервис, ответ. Обязательность и формат query-параметров — через
`XxxQuerySerializer(data=request.query_params).is_valid(raise_exception=True)`, не `if` во view. Вычисляемые
поля списка — метод сервиса, навешивающий атрибут, не цикл во view. Пагинация остаётся во view.

## Доступ на запись — в `get_queryset`

Запрет трогать часть видимых объектов — в `get_queryset` для `update`/`partial_update`/`destroy`, не в
`Serializer.validate`: DELETE не проходит через сериализатор. Недоступный объект выпадает → `get_object()`
отдаёт 404. Фильтр — в сервисе, эталон `UserService.get_editable_queryset`. Проверки полей (нельзя менять
себе роль) остаются в сериализаторе. Источник: MCA-692.

## Разделять по action — при возможности

Отдельные ViewSet'ы, если у action различаются права, scope, сериализаторы или queryset: разные
permission-классы на чтение и запись, `get_serializer_class()` на 4+ ветки, нетривиальный workflow
(approve/decline). Одним — простой CRUD, справочники, отличия только в `read_only_fields`. Ради одного
отличия в `select_related` два класса не плодить.

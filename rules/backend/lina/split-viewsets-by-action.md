# Разделять ViewSet'ы по action/ресурсу — при возможности

**Когда применять:** при создании нового или существенной правке существующего ViewSet.

**Правило:** если у разных action одного ресурса различаются права/scope/сериализаторы/queryset — **по возможности** разделять на отдельные ViewSet'ы, а не запихивать всё в один `ModelViewSet`.

**Признаки, что пора разделять:**

- Разный scope. Пример: `TimeClockListViewSet` (без `is_primary`) и `TimeClockUpdateViewSet` (с `is_primary=True`).
- Разные permission-классы для чтения и записи (через `get_permissions()` уже плохо читается).
- Разные `select_related`/`prefetch_related` для list и retrieve, при этом retrieve содержит нетривиальную логику.
- Сложный `get_serializer_class()` с 4+ ветками — кандидат на разделение.
- Один action имеет нетривиальный workflow (claim/approve/decline) — кастомные `@action` уже на грани, лучше отдельный ViewSet.

**Когда оставлять одним ViewSet:**

- Простой CRUD без различий в правах/scope (справочники).
- Все action ходят в один сервис без специализации.
- Сериализаторы отличаются только полями для чтения/записи (`read_only_fields` хватает).

**Пример:**

```python
# Хорошо — два отдельных ViewSet'а
class TimeClockListViewSet(ScopedViewSetMixin, GenericViewSet, ListModelMixin, RetrieveModelMixin):
    queryset = TimeClock.objects.all()

    def get_team_queryset(self, qs):
        return filter_by_outlet(user=self.request.user, queryset=qs, field_name='outlet_id')


class TimeClockUpdateViewSet(ScopedViewSetMixin, GenericViewSet, UpdateModelMixin):
    queryset = TimeClock.objects.all()
    http_method_names = ['put', 'patch']

    def get_team_queryset(self, qs):
        return filter_by_outlet(
            user=self.request.user, queryset=qs, field_name='outlet_id', is_primary=True,
        )
```

**Почему:** разделение упрощает чтение, защищает от регрессий (правка одного action не задевает другие), декларативно показывает, что у ресурса допустимо. Один большой ViewSet с условиями по `self.action` копит ветвления и становится местом регрессий — особенно в scope-методах.

«При возможности» — потому что разделение не должно превращаться в догму: ради одного отличия в `select_related` плодить два класса не нужно.

## Связанное

- [[viewset-patterns]] — 4 типа ViewSet и когда какой
- [[is-primary-for-sensitive-scope]] — пример, где разный scope для read и write

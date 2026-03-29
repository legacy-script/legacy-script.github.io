---
title: DRY CRUD в Django
draft: false
---

## Как реализовать CRUD в Django с минимальным повторением кода

При разработке веб-приложения часто возникает задача: нужно реализовать базовые операции (CRUD — create, read, update, delete) для одной или нескольких моделей.

В Django для этого есть элегантный инструмент — [generic class-based views](https://docs.djangoproject.com/en/6.0/topics/class-based-views/generic-display/). С их помощью можно реализовать представление (view), написав всего несколько строк кода.

Сложности возникают, когда представления ссылаются друг на друга. Например, зачастую каждый элемент в списке (list view) является ссылкой на страницу деталей (detail view), которая, в свою очередь, содержит кнопки для перехода к редактированию (update view) или возврата к списку. Представление удаления (delete view) после выполнения операции обычно перенаправляет пользователя обратно к списку, и так далее. Реализация всех этих связей для каждой модели — процесс весьма утомительный.

В Django Rest Framework для упрощения создания связанных представлений был введён класс [ViewSet](https://www.django-rest-framework.org/api-guide/viewsets/). В этой статье я покажу, как реализовать аналогичную идею не для REST-приложения, а для проекта, построенного по классической архитектуре Model-View-Template (MVT).

### Что мы будем делать

В качестве примера рассмотрим приложение для управления данными о сотрудниках компании. Необходимо реализовать CRUD для модели Employee:

```python
class Employee(models.Model):
    employee_id = models.BigAutoField(verbose_name=_("Unique ID"), primary_key=True)
    first_name = models.CharField(verbose_name=_("First name"), max_length=20)
    last_name = models.CharField(verbose_name=_("Last name"), max_length=25)
    email = models.CharField(verbose_name=_("E-mail"), max_length=100)
    phone_number = models.CharField(verbose_name=_("Phone number"), null=True, max_length=20)
    hire_date = models.DateField(verbose_name=_("Hire date"))
    job = models.ForeignKey("Job", on_delete=models.CASCADE)
    salary = models.DecimalField(verbose_name=_("Salary"), max_digits=8, decimal_places=2)
    manager = models.ForeignKey("Employee", null=True, on_delete=models.CASCADE)
    department = models.ForeignKey("Department", on_delete=models.CASCADE)
```

Здесь поля `job` и `department` являются ссылками на модели `Job` и `Department` соответственно, а поле `manager` — ссылкой на саму модель `Employee`. При редактировании значения для этих полей должны выбираться из выпадающего списка. Список (list view) должен представлять собой таблицу с фильтрами.

Всё вместе это должно выглядеть так:
{{<youtube id="ch9SzmsQA3E">}}

### Инструменты реализации

Для реализации я использовал следующие пакеты:

* [django-tables2](https://django-tables2.readthedocs.io/en/latest/) — с его помощью отрисовывается таблица в списке,
* [django-filter](https://django-filter.readthedocs.io/en/stable/) — для создания фильтров в таблице,
* [django-crispy-forms](https://django-crispy-forms.readthedocs.io/en/latest/) и [crispy-bootstrap5](https://pypi.org/project/crispy-bootstrap5/) нужны для форм редактирования (приложение использует Bootstrap 5),
* [django-select2](https://django-select2.readthedocs.io/en/stable/) необходим для выпадающих списков в полях, ссылающихся на другие модели.

Полную реализацию можно посмотреть на [GitHub](https://github.com/dpdevotee/dry-cruds/tree/master).

### Архитектура TableViewSet

Я не буду подробно разбирать каждый элемент реализации, а лишь опишу общую идею и неочевидные моменты.

Основной класс, инкапсулирующий все необходимые представления, называется `TableViewSet` ([common/viewset.py::TableViewSet](https://github.com/dpdevotee/dry-cruds/blob/master/common/viewset.py#L16)).

Конструктор этого класса принимает следующие аргументы:

* `model` — модель, для которой создаётся CRUD;
* `table_class` — класс таблицы для списка (используется [django-tables2](https://django-tables2.readthedocs.io/en/latest/));
* `filterset_class` — класс фильтров для таблицы (используется [django-filter](https://django-filter.readthedocs.io/en/stable/));
* `form_class` — класс формы для редактирования экземпляра модели;
* `base_url_pattern` — строка-префикс для всех эндпоинтов в CRUD (эндпоинты имеют вид `{base_url_pattern}/`, `{base_url_pattern}/create/`, `{base_url_pattern}/<id>/detail/`, `{base_url_pattern}/<id>/update/`, `{base_url_pattern}/<id>/delete/`);
* `base_url_name` — строка-префикс для всех имён эндпоинтов в CRUD (имена используются внутри шаблонов).

Пример использования:

```python
from django_filters import CharFilter, FilterSet
from django_tables2 import Table
from common.filters import DateFromToRangeFilter, RangeFilter
from common.viewset import TableViewSet
from .forms import EmployeeForm
from .models import Employee


class EmployeeTable(Table):
    class Meta:
        model = Employee
        template_name = "django_tables2/bootstrap5.html"
        fields = (
            "first_name",
            "last_name",
            "email",
            "phone_number",
            "hire_date",
            "job",
            "salary",
            "manager",
            "department",
        )


class EmployeeFilter(FilterSet):
    first_name = CharFilter(lookup_expr="icontains", label="First name")
    last_name = CharFilter(lookup_expr="icontains", label="Last name")
    email = CharFilter(lookup_expr="icontains", label="Email")
    phone_number = CharFilter(lookup_expr="icontains", label="Phone number")
    hire_date = DateFromToRangeFilter(label="Hire date")
    salary = RangeFilter(label="Salary")
    job = CharFilter(field_name="job__job_title", lookup_expr="icontains", label="Job")
    manager = CharFilter(field_name="manager__last_name", lookup_expr="icontains", label="Manager")
    department = CharFilter(field_name="department__department_name", lookup_expr="icontains", label="Department")

    class Meta:
        model = Employee
        fields = []


employee_viewset = TableViewSet(
    model=Employee,
    table_class=EmployeeTable,
    filterset_class=EmployeeFilter,
    base_url_pattern="employees",
    base_url_name="employees",
    form_class=EmployeeForm,
)
```

Экземпляр этого класса имеет атрибут `urls`, который можно добавить в `urlpatterns`, и в приложении появятся представления для всех CRUD-операций:

```python
from django.urls import include, re_path
from .views import employee_viewset

urlpatterns = [
    re_path(r"", include(employee_viewset.urls)),
    # другие пути
]
```

Класс `TableViewSet` динамически создаёт представление для каждой операции. Вот пример того, [как создаётся list view](https://github.com/dpdevotee/dry-cruds/blob/master/common/viewset.py#L54):

```python
    def _build_list_url(self):
        view_set = self

        class TableWithLinks(self.table_class):
            pk = Column(verbose_name="ID", accessor="pk", orderable=False, linkify=(self.detail_url_name, (A("pk"),)))

            class Meta:
                template_name = "django_tables2/bootstrap5.html"
                sequence = ("pk", "...")

        class NewView(SingleTableMixin, FilterView):
            model = self.model
            table_class = TableWithLinks
            filterset_class = self.filterset_class
            template_name = "common/viewsets/list.html"
            paginate_by = 10
            ordering = "pk"
            header = self.model._meta.verbose_name_plural.capitalize()

            def get_links(self):
                return [
                    PrimaryButtonLink(
                        text="Create",
                        url_name=view_set.create_url_name,
                    ),
                ]

        return re_path(f"^{self.base_url_pattern}/$", NewView.as_view(), name=self.list_url_name)
```

Обратите внимание, что у класса `NewView` есть метод `get_links`. Он возвращает объекты ссылок, с помощью которых представления для разных CRUD-операций связаны между собой в шаблонах. Вот так выглядит [шаблон для list view](https://github.com/dpdevotee/dry-cruds/blob/master/templates/common/viewsets/list.html):

```django
{% extends 'common/base.html' %}
{% load i18n l10n %}
{% load crispy_forms_tags %}
{% load django_tables2 %}

{% block content %}
  <h1 class="mt-5 mb-4"> {{ view.header }} </h1>
  <div class="table-container">
    <form method="get">
      <div class="card">
        <div class="card-body">
          <div class="row g-3">
            {% for field in filter.form %}
            <div class="col-2">
              {{ field|as_crispy_field }}
            </div>
            {% endfor %}
          </div>
          <div class="row g-3">
            <div class="col-2">
              <div class="mb-3">
                <input type="submit" class="btn btn-primary" value="Submit"/>
              </div>
            </div>
          </div>
        </div>
      </div>
    </form>
    {% render_table table %}
  </div>

  {% for button in view.get_links %}
    {{ button }}
  {% endfor %}
{% endblock %}
```

Также стоит обратить внимание на то, что первая колонка в таблице — это колонка со ссылками на detail view для каждой строки. Это реализовано с помощью класса `TableWithLinks`: он наследуется от исходного класса таблицы и добавляет новую колонку `pk`, которая благодаря параметру `linkify=(self.detail_url_name, (A("pk"),))` становится кликабельной.

[Метод](https://github.com/dpdevotee/dry-cruds/blob/master/common/viewset.py#L183) `urls` класса `TableViewSet` просто последовательно создаёт URL для каждой операции и возвращает их список:

```python
    @property
    def urls(self):
        return [
            self._build_list_url(),
            self._build_detail_url(),
            self._build_create_url(),
            self._build_update_url(),
            self._build_delete_url(),
        ]
```

И последнее. Для того чтобы поля выбора `job`, `department` и `manager` сотрудника стали выпадающими списками, необходимо [добавить](https://github.com/dpdevotee/dry-cruds/blob/master/dry_cruds/urls.py#L23) в `urlpatterns` поддержку `select2`:

```python
from django.contrib import admin
from django.urls import include, path, re_path

urlpatterns = [
    path("admin/", admin.site.urls),
    re_path(r"", include("hr.urls")),
    # select2 создает эндпоинты, по которым Javascript
    # получает значения для выпадающих списков
    path("select2/", include("django_select2.urls")),
]
```

Также необходимо [определить виджеты](https://github.com/dpdevotee/dry-cruds/blob/master/hr/forms.py) для этих полей в формах:

```python
from django import forms
from django_select2 import forms as s2forms

from .models import Employee, Job


class JobWidget(s2forms.ModelSelect2Widget):
    search_fields = [
        "job_title__icontains",
    ]


class ManagerWidget(s2forms.ModelSelect2Widget):
    search_fields = [
        "first_name__icontains",
        "last_name__icontains",
    ]


class DepartmentWidget(s2forms.ModelSelect2Widget):
    search_fields = [
        "department_name__icontains",
    ]


class EmployeeForm(forms.ModelForm):
    class Meta:
        model = Employee
        fields = "__all__"
        widgets = {
            "job": JobWidget,
            "manager": ManagerWidget,
            "department": DepartmentWidget,
        }
```

Теперь, когда мы знаем, как реализовать CRUD для модели `Employee`, легко сделать то же самое и для других моделей приложения. При этом нам не придётся писать новые шаблоны и беспокоиться о связях между представлениями — всё, что нужно сделать, это определить таблицу, фильтры и форму.

### Заключение

Итак, мы разобрали, как реализовывать CRUD в Django, придерживаясь принципа DRY. Мы не затрагивали тему контроля доступа, но ей стоит посвятить отдельную статью.

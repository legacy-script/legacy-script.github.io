---
title: DRY CRUD in Django
draft: false
---

## How to implement CRUD in Django with minimal code repetition

When developing a web application, a common task arises: you need to implement basic operations (CRUD — create, read, update, delete) for one or more models.

Django provides an elegant tool for this — [generic class-based views](https://docs.djangoproject.com/en/6.0/topics/class-based-views/generic-display/). They allow you to create a view by writing just a few lines of code.

Difficulties arise when views reference each other. For example, often each item in a list (list view) is a link to a detail page (detail view), which, in turn, contains buttons to navigate to editing (update view) or returning to the list. A delete view, after performing the operation, usually redirects the user back to the list, and so on. Implementing all these links for each model is a rather tedious process.

In Django Rest Framework, the [ViewSet](https://www.django-rest-framework.org/api-guide/viewsets/) class was introduced to simplify the creation of related views. In this article, I will show how to implement a similar idea, not for a REST application, but for a project built using the classic Model-View-Template (MVT) architecture.

### What we are going to do

As an example, let's consider an application for managing company employee data. We need to implement CRUD for the `Employee` model:

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

Here, the `job` and `department` fields are links to the `Job` and `Department` models respectively, and the `manager` field is a link to the `Employee` model itself. When editing, the values for these fields should be selected from a dropdown list. The list (list view) should be a table with filters.

Altogether, it should look like this:
{{<youtube id="ch9SzmsQA3E">}}

### Implementation Tools

I used the following packages for the implementation:

* [django-tables2](https://django-tables2.readthedocs.io/en/latest/) — used to render the table in the list,
* [django-filter](https://django-filter.readthedocs.io/en/stable/) — for creating filters in the table,
* [django-crispy-forms](https://django-crispy-forms.readthedocs.io/en/latest/) and [crispy-bootstrap5](https://pypi.org/project/crispy-bootstrap5/) are needed for the editing forms (the application uses Bootstrap 5),
* [django-select2](https://django-select2.readthedocs.io/en/stable/) is required for dropdown lists in fields referencing other models.

The full implementation can be found on [GitHub](https://github.com/dpdevotee/dry-cruds/tree/master).

### TableViewSet Architecture

I won't go into detail on every element of the implementation, but I'll describe the general idea and non-obvious points.

The main class encapsulating all the necessary views is called `TableViewSet` ([common/viewset.py::TableViewSet](https://github.com/dpdevotee/dry-cruds/blob/master/common/viewset.py#L16)).

The constructor of this class takes the following arguments:

* `model` — the model for which the CRUD is created;
* `table_class` — the table class for the list (uses [django-tables2](https://django-tables2.readthedocs.io/en/latest/));
* `filterset_class` — the filter class for the table (uses [django-filter](https://django-filter.readthedocs.io/en/stable/));
* `form_class` — the form class for editing a model instance;
* `base_url_pattern` — the URL prefix for all endpoints in the CRUD (endpoints look like `{base_url_pattern}/`, `{base_url_pattern}/create/`, `{base_url_pattern}/<id>/detail/`, `{base_url_pattern}/<id>/update/`, `{base_url_pattern}/<id>/delete/`);
* `base_url_name` — the name prefix for all endpoint names in the CRUD (names are used inside templates).

Usage example:

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

An instance of this class has a `urls` attribute that can be added to `urlpatterns`, and views for all CRUD operations will appear in the application:

```python
from django.urls import include, re_path
from .views import employee_viewset

urlpatterns = [
    re_path(r"", include(employee_viewset.urls)),
    # other paths
]
```

The `TableViewSet` class dynamically creates a view for each operation. Here is an example of how the list view is created:

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

Note that the `NewView` class has a `get_links` method. It returns link objects used to connect views for different CRUD operations within the templates. This is what the list view template looks like:

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

Also, notice that the first column in the table is a column with links to the detail view for each row. This is implemented using the `TableWithLinks` class: it inherits from the original table class and adds a new `pk` column, which becomes clickable thanks to the `linkify=(self.detail_url_name, (A("pk"),))` parameter.

The `urls` [method](https://github.com/dpdevotee/dry-cruds/blob/master/common/viewset.py#L183) of the `TableViewSet` class simply sequentially creates a URL for each operation and returns their list:

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

And finally. To make the selection fields for `job`, `department`, and the employee's `manager` into dropdown lists, you need to [add](https://github.com/dpdevotee/dry-cruds/blob/master/dry_cruds/urls.py#L23) `select2` support to `urlpatterns`:

```python
from django.contrib import admin
from django.urls import include, path, re_path

urlpatterns = [
    path("admin/", admin.site.urls),
    re_path(r"", include("hr.urls")),
    # select2 creates endpoints from which Javascript
    # gets values for dropdown lists
    path("select2/", include("django_select2.urls")),
]
```

You also need to [define widgets](https://github.com/dpdevotee/dry-cruds/blob/master/hr/forms.py) for these fields in the forms:

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

Now that we know how to implement CRUD for the `Employee` model, it's easy to do the same for other models in the application. You won't have to write new templates or worry about the links between views — all you need to do is define the table, filters, and form.

### Conclusion

In this article, we've explored how to implement CRUD in Django while following the DRY principle. We haven't touched upon access control, but that topic deserves its own separate article.

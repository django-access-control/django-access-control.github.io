---
title: "Userguide"
date: 2021-06-22
weight: 1
description: >
  Here is the quick overview how to use the framework.
---

## The theory

<blockquote>
Experience by itself teaches nothing...<br>
Without theory, experience has no meaning.<br>
Without theory, one has no questions to ask.<br>
Hence without theory there is no learning.<br>
~ <cite>W. Edwards Deming</cite>
</blockquote>

Yes, theories can be boring. However, we need to establish some common vocabulary before we can move forward. Therefore, please review the [theory](/docs/theory/) section before heading on to practical stuff.

## Installation

```sh
pip install django-access-control
```

That's it. No need to add anything to `INSTALLED_APPS`, configure the `AUTHENTICATION_BACKENDS` or inherit from our `User` model. Also, we do not create any objects in your database.


## Implementing access control

Different systems need different granularity when it comes to access control.
Let's go over some examples moving from more broad categories to more specific ones.

We will implement a simplified version of [the example project](/docs/example-project/): a Q&A site.
Let's create the model `Question` as follows:

```py
from django.db import models

class Question(models.Model):
    title = models.CharField(max_length=100)
    body = models.TextField()
    author = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.PROTECT)
    is_published = models.BooleanField(default=True)

    objects = QuestionQuerySet.as_manager()

    def __str__(self) -> str:
        return self.title
```

Above the model, let's add the query set:

```py
from django_access_control.querysets import ConfidentialQuerySet

class QuestionQuerySet(ConfidentialQuerySet):
    pass
```

Finally, let's register oue model to the DJango admin site:

```py
from django.contrib import admin
from django_access_control.admin import ConfidentialModelAdmin

from questions.models import Question


@admin.register(Question)
class QuestionAdmin(ConfidentialModelAdmin):

    def save_model(self, request, obj, form, change):
        """
        Automatically fill `question.author` with the current user
        when a new question is added.
        """
        if not change:  # `not change` means the obj is added, not modified
            obj.author = request.user
        super().save_model(request, obj, form, change)
```

Now everything should work in the same way as it normal would since our permission policy defaults are the same as Django's to make it more intuitive to use.

### Table level permissions

There are four methods to configure table level permissions. Here they are with their default implementations:

```py
def has_table_wide_add_permission(self, user: AbstractUser) -> bool:
    return user.is_superuser or has_permission(user, "add", self.model)

def has_table_wide_view_permission(self, user: AbstractUser) -> bool:
    return user.is_superuser or has_permission(user, "view", self.model)

def has_table_wide_change_permission(self, user: AbstractUser) -> bool:
    return user.is_superuser or has_permission(user, "change", self.model)

def has_table_wide_delete_permission(self, user: AbstractUser) -> bool:
    return user.is_superuser or has_permission(user, "delete", self.model)
```

Let's override one of them to give every logged in user the right to add questions:

```py
def has_table_wide_add_permission(self, user: AbstractUser) -> bool:
    return user.is_authenticated

```

### Row level permissions

If the user has some permission for the entire table, it will apply to all the rows. The methods below will give you more granular control by allowing to you give extra permission to a limited number or rows.

The methods you can use are:

```py
def rows_with_extra_view_permission(self, user: AbstractUser) -> QuerySet:
    return self.none()

def rows_with_extra_change_permission(self, user: AbstractUser) -> QuerySet:
    return self.none()

def rows_with_extra_delete_permission(self, user: AbstractUser) -> QuerySet:
    return self.none()
```

By default they do not grant permission to any rows. Let's adjust the `view` permissions so that 

* everybody can see published questions
* staff members can see all questions
* authors can see their questions even if they are not published

```py
def rows_with_extra_view_permission(self, user: AbstractUser) -> QuerySet[Question]:
    if user.is_staff:
        return self
    return self.filter(is_published=True) | \
            (self.filter(author=user) if user.is_authenticated else self.none())
```

On the last line `|` is the bitwise OR operator which returns the union of the two querysets (this is why we learned set theory at school 🙂 ).

The `if user.is_authenticated else` is necessary because if the user is not logged in, they are represented by an [`AnonymousUser`](https://docs.djangoproject.com/en/dev/ref/contrib/auth/#django.contrib.auth.models.AnonymousUser) which has a more limited interface compared to the regular `User`. Without this check, `AnonymousUser` would be passed to `self.filter(author=user)` which would result in an exception in the lower layers (which was pretty hard to track down...).

{{% alert title="Be mindful!" %}}
There are two methods with very similar names:

* `rows_with_extra_view_permission`
* `rows_with_view_permission`

You should override the first one. The later on will use `rows_with_extra_view_permission` as well as `has_table_wide_view_permission` to combine together all the rows the user can view. That method might be handy if you need to fetch all the rows in your view layer. It is also internally used in the Django admin integration.
{{% /alert %}}

### Colum level permissions

There are three methods for fine-tuning field level permissions:

```py
@classmethod
def addable_fields(cls, user: AbstractUser) -> FrozenSet[str]:
    """
    Control which fields the user can specify when creating a new object (row).
    NB! Make sure that all not nullable fields that are not specified here
    would be populated automatically
    to prevent IntegrityError for the database.
    """
    return frozenset(all_field_names(cls.model))

@staticmethod
def viewable_fields(user: AbstractUser, obj) -> FrozenSet[str]:
    """
    Control which wields the user can view.
    """
    return frozenset(all_field_names(obj))

@staticmethod
def changeable_fields(user: AbstractUser, obj) -> FrozenSet[str]:
    """
    Control which wields the user can edit.
    """
    return frozenset(all_field_names(obj))
```

By default the user has access to all the fields. Let's update it so that

* the question authors can update the `body` of the question
* stuff members can publish and unpublish questions
* superusers can do everything

```py
@staticmethod
def changeable_fields(user: AbstractUser, obj: Question) -> FrozenSet[str]:
    if user.is_superuser: return frozenset(all_field_names(obj))
    fields = frozenset()
    if obj.author == user: fields |= frozenset({"body"})
    if user.is_staff: fields |= frozenset({"is_published"})
    return fields
```
{{% alert %}}
We use `frozenset` -- an immutable version of the regular `set` -- because immutability greatly reduces the risk of human error (such as passing the mutable set to a function as an argument, modifying it in the function and not being aware that the changes will also influence other places where the same set is used). And mistakes in the security layer can be costly...
{{% /alert %}}

## The view layer

Django Access Control provides out-of-the-box integration with the Django admin site through the `ConfidentialModelAdmin` class. You can use its implementation as an example for your custom views and forms as well.

In addition to the methods already discussed above, `ConfidentialQuerySet` provides some special methods intended to be used in the view layer:

* `rows_with_view_permission(self, user: AbstractUser) -> QuerySet`
* `rows_with_change_permission(self, user: AbstractUser) -> QuerySet`
* `rows_with_delete_permission(self, user: AbstractUser) -> QuerySet`
* `rows_with_some_permission(self, user: AbstractUser) -> QuerySet` -- returns all rows for which the user has at least some kind of permissions
* `has_some_permissions(self, user: AbstractUser) -> bool` -- indicates whether the user has any permission to any instance of the model
* `contains(self, obj: Model) -> bool` -- indicates whether the given queryset contains the `obj`

{{% alert color="warning" %}}
Django Access Control cannot guard you from everything.<br>
Please refer to [the security notice](/docs/security-notice/) for further details.
{{% /alert %}}
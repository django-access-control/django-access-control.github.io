---
title: "Userguide"
date: 2021-06-22
weight: 1
description: >
  Here is the quick overview how to use the framework.
---

## What are permissions

A permission consists of an *action* and an *object*.
{{% light %}}E.g. "can view *(action)* question with ID 5 *(object)*."{{% /light %}}

Permissions are hierarchical: their object can be

* an entire database **table** (a Django model), {{% light %}}e.g. "can view all questions"{{% /light %}}.
* a specific **row** (an instance of a Django model), {{% light %}}e.g. "can view all question with ID 5"{{% /light %}}.
* a specific **field** of a specific row (a field of an instance of a Django model), {{% light %}}e.g. "can view the title of the question with ID 5"{{% /light %}}.

If one has the permission "can view all questions", they implicitly also have "can view all question with ID 5" and "can view the title of the question with ID 5".

We refer to this as the *scope* of the permission.

Permission actions should theoretically be hierarchical as well, but we have not found a usecase for it yet.

Currently we support the same four permission actions as Django natively does:

* add
* view
* change
* delete

Some actions are restricted to a specific scope. For instance, you can have the permission "can add a question" (with table scope), but "can add a question with ID 5" would make no sense given that such a question already exists. Thus the "add" permission always requires the "table" scope. Likewise, the "delete" permission can be granted for all rows in a table or for a specific row, but not for a field.

|        | table/model | row/object | field |
|--------|-------------|------------|-------|
| add    | X           | -          | -     |
| delete | X           | X          | -     |
| view   | X           | X          | X     |
| change | X           | X          | X     |


## Implementing the permission system

Different systems need different granularity when it comes to access control.
Let's go over some examples. 

```py
class ConfidentialQuerySet(QuerySet):

    # Table wide permissions

    def has_table_wide_add_permission(self, user) -> bool:
        return user.is_superuser or has_permission(user, self.model, "add")
    
    def has_table_wide_view_permission(self, user) -> bool:
        return user.is_superuser or has_permission(user, self.model, "view")

    def has_table_wide_change_permission(self, user) -> bool:
        return user.is_superuser or has_permission(user, self.model, "change")

    def has_table_wide_delete_permission(self, user) -> bool:
        return user.is_superuser or has_permission(user, self.model, "delete")

    # Row permissions

    def rows_with_view_permission(self, user) -> QuerySet:
        return self if self.has_table_wide_view_permission(user) else self.none()

    def rows_with_change_permission(self, user) -> QuerySet:
        return self if self.has_table_wide_change_permission(user) else self.none()

    def rows_with_delete_permission(self, user) -> QuerySet:
        return self if self.has_table_wide_delete_permission(user) else self.none()

    # Column permissions

    def columns_with_view_permission(self, user, only=("pk",)) -> Dict[Model, FrozenSet[str]]:
        return {obj: all_fields(obj) for obj in self.only(*only).rows_with_view_permission(user)}

    def columns_with_change_permission(self, user, only=("pk",)) -> Dict[Model, FrozenSet[str]]:
        return {obj: all_fields(obj) for obj in self.only(*only).rows_with_change_permission(user)}


class QuestionQuerySet(ConfidentialQuerySet):

    # Table wide permissions

    def has_table_wide_add_permission(self, user) -> bool:
        return user.is_authenticated and user.reputation >= 5
    
    def has_table_wide_view_permission(self, user) -> bool:
        return True

    # Use default logic for table wide change and delete permissions

    # Row permissions

    # No need to grant view permissions to specific rows as they are already public.
    #  Thus the default implementation is kept.

    def rows_with_change_permission(self, user) -> QuerySet:
        return super().rows_with_change_permission(user) | self.filter(author=user)

    # Column permissions

    def columns_with_change_permission(self, user, obj, only=("pk",)) -> Dict[Model, FrozenSet[str]]:
        extra_fields = {obj: frozenset('needs_review') for obj in self.only(*only) if user.reputation >=20 else {}
        return extra_fields | super().columns_with_change_permission(user, obj)

```
---
title: "Notes on security"
date: 2021-067-08
weight: 4
---

Even if you use Django Access Control, you need to pay attention to some things we cannot guard you against.

Firstly, we provide the `QuerySet` methods for enforcing access control, but it is up to you to use them in your views. If you accidentally have `return Question.objects.all()` instead of `return Question.objects.rows_with_view_permission(user)`, you bypass all the security checks.


## Django admin integration

Beware that:

* **Field level permissions are not enforced for list view** since the same fields need to be shown for all the rows.
* You need to **enforce field level permissions in [admin actions](https://docs.djangoproject.com/en/dev/ref/contrib/admin/actions/)** yourself: since they are functions defined by you, you are responsible for the security as well.
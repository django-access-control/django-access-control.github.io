---
title: "Theory"
date: 2021-06-22
weight: 1
description: >
  Here we define the terminology we use throughout the project and provide a little background regarding the decisions made in this project.
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

Currently we support the same four permission actions as Django natively does:

* add
* view
* change
* delete

Some actions are restricted to a specific scope. For instance, you can have the permission "can add a question" (with table scope), but "can add a question with ID 5" would make no sense given that such a question already exists. Likewise, the "delete" permission can be granted for all rows in a table or for a specific row, but not for a field (updating the value of a field to be `None`/`null` would not quality as deleting it).

|        | table/model | row/object | field |
|--------|-------------|------------|-------|
| add    | X           | -          | X     |
| delete | X           | X          | -     |
| view   | X           | X          | X     |
| change | X           | X          | X     |

## Access control system in software layers

A good software system consists of distinct layers. Although the layers themselves are usually the same, each framewrok names them differently. In case of Django, they would be:


<table id="system-layers-diagram">
  <tr>
      <td style="background-color: #85f56c">Template</td>
      <td rowspan=2 style="background-color: #ffed78">Django admin</td>
  </tr>
  <tr>
      <td style="background-color: #f27dff">View</td>
  </tr>
  <tr>
      <td colspan=2 style="background-color: #a07dff">Business logic</td>
  </tr>
  <tr>
      <td colspan=2 style="background-color: #4287f5">Data access</td>
  </tr>
</table>

<style>
  #system-layers-diagram td {
    text-align: center; 
    vertical-align: middle;
  }
</style>


* the template layer (serializer layer in REST APIs) -- responsible for formating the response the user can see
* the view layer -- responsible for the content the user can see
* the service layer -- responsible for an individual unit of work which can be used by many views. In smaller projects it is OK to merge it with the view layer, but in bigger projects a distinct service layer makes it easier to reuse, decouple and test your code
* the data access layer -- responsible for retrieving and saving the data, in other words, responsible for communicating with the underlying database

The Django admin site combines the template and the view layers (although you do can dig in and modify both the underling templates and views for more granular customizations).

It is technically possible to implement the access control system in any of the layers. You can even put it into the template layer -- meaning that your security mechanisms live alongside your HTML tags -- but this lead to numerous problems regarding both code readability as well as security risks.

The deeper in the layer system the access control system is implemented, the fewer code duplication we need and the less likely it is that we accidentally forget to consider it in some database queries.

Thus, the access control system should be implemented in the data access layer, in Django terms, in the `QuerySet`.

## How the access control is used in higher layers

The basic idea is to have have special methods on the `QuerySet` which will figure out, which data the user can access. This method will be chained with other methods performing filtering based on which data we need to retrieve, e.g. `Question.objects.rows_with_view_permission(user).filter(is_relevant=True)`.

## Explicit and implicit permissions

Permissions can be *granted explicitly* or be *deducted implicitly* from existing data. E.g. in a StackOverflow style system you can update a question if

* you are an admin (the admin privileges have been *explicitly* granted to you)
* you are the one who asked the question (your ownership position *implicitly* grants you the permission)
* you have enough reputation (your high reputation *implicitly* grants you the permission)

Thus the access control methods in the `QuerySet` have the entire database at their disposal: they can check for entries in the dedicated permissions table(s) as well as look for other pieces of data to deduct permissions from.

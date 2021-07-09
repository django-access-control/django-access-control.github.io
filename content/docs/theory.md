---
title: "Theory"
date: 2021-06-22
weight: 1
description: >
  Here we define the therminology we use throughout the project and explain the mathematics behind the permissions system.
---

## Permissions

A permission indicates that somebody can to do some action with something. Thus a permission usually consists of a subject, an object and an action.

E.g: A logged in user (subject) can ask (action) a question (object).

In the context of software systems, however, we usually define the permission as action + object and then assign this permission some group of users. In this case the permissions would be "can ask a question" and it can be granted to all the logged in users.

## Permissions in the relational model

Permissions can be hierarchical: the permission "can view the title of question 5" is included in "can view question 5" which in turn is included in "can view all questions".

The above example shows how permission can have different scopes. In the context of the relational data model, we can distinguish

* tables
* individual rows
* individual columns of the row

## The actions

In a simple case we only need four types of actions:

* **add** a new row
* **view** an existing row/column
* **change** an existing row/column
* **delete** an existing row

As seen from the example, not all actions are applicable for all scopes. For example: it makes sense to give a user the permission to add a new row, but not to add a new column to an existing row, because in the relational model, rows have a fixed set of columns.

The four actions and their scopes are summarized in the following table where `X` indicates a positive and `-` a negative value:

|        | table/model | row/object | field |
|--------|-------------|------------|-------|
| add    | X           | -          | -     |
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
* the service layer -- responsible for an individual unit of work which can be used by many views. In smaller projects it is OK to merge it with the view layer, but in bigger projects it makes it easier to reuse, decouple and test your code
* the data access layer -- responsible for retrieving and saving the data, in other words, responsible for communicating with the underlying database

The Django admin site combines the template and the view layers (although you do can dig in and modify both the underling templates and views for more granular customizations).

It is technically possible to implement the access control system in any of the layers. You can even put it into the template layer -- meaning that your security mechanisms live alongside your HTML tags -- but this lead to numerous problems regarding both code readability as well as security risks.

The deeper in the layer system the access control system is implemented, the fewer codu duplication we need and the less likely it is that we accidentally forget to consider it in some database query.

Thus, the access control system should be implemented in the data access layer, in Django terms, in the `QuerySet`s.

## How the access control is used in higher layers

The basic idea is to have have special methods on the `QuerySet` which will figure out, which data the user can access. This method will be chained with other methods performing filtering based on which data we need to retrieve, e.g. `Question.objects.filter(is_relevant=True).can_view(user)`.

## Explicit and implicit permissions

Permissions can be *granted explicitly* or be *deducted implicitly* from existing data. E.g. in a StackOverflow style system you can update a question if

* you are an admin (the admin privileges have been *explicitly* granted to you)
* you are the one who asked the question (your ownership position *implicitly* grants you the permission)
* you have enough reputation (your high reputation *implicitly* grants you the permission)

Thus the access control methods in the `QuerySet` have the entire database at their disposal: they can check for entries in the dedicated permissions table(s) as well as look for other pieces of data to deduct permissions from.

---

* This implementation is good because we do not sneak in too deep. You do not have to use our special User model or authentication backend. Our solution does not alter your database. Your settings file does not need to know anything about it.

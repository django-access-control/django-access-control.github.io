---
title: "Changelog"
date: 2021-067-08
weight: 4
description: >
  All notable changes to this project will be documented here.

  The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
  and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
---

## 1.0 - {{% light %}}2021-07-09{{% /light %}}

### Changed

* Enhance `ConfidentialQuerySet` API
* Enhance integration with DJango admin site


## 0.4 - {{% light %}}2021-06-24{{% /light %}}

### Changed

* Remove the need to use a custom authentication backend
* Enhance `ConfidentialQuerySet` API
* Enhance integration with DJango admin site


## 0.3 - {{% light %}}2021-06-24{{% /light %}}

### Changed

* Make access control hierarchical, introduce the idea of quantifiers
* Refactor access control logic out of the `ModelAdmin` class so that is could be used in other places as well

## 0.2 - {{% light %}}2021-06-24{{% /light %}}

### Changed

* Project was renamed from Django Granular Permissions to Django access Control

## 0.1 - {{% light %}}2021-06-23{{% /light %}}

### Added

* Added `ConfidentialQuerySet` which provides access control on the data access level
* Added `ConfidentialModelAdmin` which can use `ConfidentialQuerySet` methods to enforce access control in Django admin
* Added `AuthenticationBackend` to handle custom permissions

---
layout: page
title: Python Course
---

<div class="hero">
  <img src="{{ '/assets/images/hero.svg' | relative_url }}" alt="Stylized code window with a Python snippet" />
</div>

# Python Course

A slide-deck-driven course covering Python, web development with Django and FastAPI, testing, tooling, and a few extras.

## Python language

- [Introduction](/slides/introduction.html)
  <br><small>What we'll cover, why Python, course structure.</small>
- [Python basics](/slides/python-basics.html)
  <br><small>Types, control flow, functions, comprehensions, exceptions.</small>
- [Object-oriented programming](/slides/oop.html)
  <br><small>Classes, inheritance, dunders, dataclasses, ABCs, Protocols.</small>
- [Metaprogramming](/slides/metaprogramming.html)
  <br><small>Decorators, descriptors, metaclasses, `__init_subclass__`.</small>
- [Refactoring Python](/slides/refactoring-python.html)
  <br><small>Code smells, SOLID, idiomatic improvements.</small>
- [Design patterns](/slides/patterns.html)
  <br><small>Why half of them disappear in Python.</small>

## Tooling & testing

- [Project setup with uv](/slides/python-tooling.html)
  <br><small>Install Python, scaffold projects, manage deps and tools — `uv`, the modern default.</small>
- [Python packages — under the hood](/slides/python-packages.html)
  <br><small>What `uv` abstracts: `pip`, `venv`, `pyproject.toml`, publishing to PyPI.</small>
- [Task runners](/slides/task-runners.html)
  <br><small>Make, `invoke`, `click`, `typer` for project commands.</small>
- [PyTest](/slides/pytest.html)
  <br><small>Fixtures, parametrize, marks, mocking, plugins.</small>
- [Static code analysis](/slides/static-code-analysis.html)
  <br><small>`ruff`, `mypy`, `pre-commit`.</small>
- [Continuous integration](/slides/continuous_integration.html)
  <br><small>Running tests, lint, and type-check on every push.</small>

## Web — Django

- [Django](/slides/django.html)
  <br><small>Project structure, settings, manage.py, the request/response cycle.</small>
- [Django models](/slides/django-models.html)
  <br><small>ORM, migrations, QuerySets, relationships, performance.</small>
- [Django URLs](/slides/django-urls.html)
  <br><small>Routing, named routes, includes, reverse lookup.</small>
- [Django views](/slides/django-views.html)
  <br><small>Function-based, class-based, generic views, mixins.</small>
- [Django templates](/slides/django-templates.html)
  <br><small>DTL, blocks, filters, forms, ModelForms.</small>
- [Django auth](/slides/django-auth.html)
  <br><small>Users, sessions, login, password hashing, allauth.</small>
- [Authorization](/slides/authorization.html)
  <br><small>Permissions, groups, object-level access, custom permission classes.</small>
- [Django apps](/slides/django-apps.html)
  <br><small>Reusable apps, signals, management commands, packaging.</small>
- [Refactoring Django](/slides/refactoring-django.html)
  <br><small>Fat models, thin views, services, custom QuerySets.</small>
- [BDD](/slides/bdd.html)
  <br><small>`pytest-bdd` and `behave` against a Django app.</small>
- [Celery](/slides/celery.html)
  <br><small>Background jobs, retries, periodic tasks, Beat.</small>
- [Django REST Framework](/slides/django-rest-framework.html)
  <br><small>Serializers, viewsets, routers, auth, throttling, JWT.</small>
- [GraphQL](/slides/graphql.html)
  <br><small>Strawberry, Graphene-Django, queries, mutations, subscriptions.</small>

## Web — FastAPI & async

- [FastAPI](/slides/fastapi.html)
  <br><small>Type-hint-driven async APIs, Pydantic, dependency injection, OpenAPI.</small>
- [WSGI & ASGI](/slides/wsgi-asgi.html)
  <br><small>The protocols Python servers and frameworks speak. gunicorn vs uvicorn.</small>
- [Async & concurrency](/slides/async-concurrency.html)
  <br><small>asyncio, threads, processes — when each pays off, when each doesn't.</small>

## Infrastructure

- [Git basics](/slides/git/basics.html)
  <br><small>Working tree, index, branches, remotes, daily commands.</small>
- [Branching models](/slides/git/gitflow.html)
  <br><small>Git Flow, GitHub Flow, trunk-based — when each pays off.</small>
- [Docker](/slides/docker/docker.html)
  <br><small>Dockerfile, layered caching, multi-stage builds, `docker compose`.</small>
- [Ansible](/slides/ansible.html)
  <br><small>Agentless YAML playbooks for server configuration.</small>
- [Deployment](/slides/deployment.html)
  <br><small>CI → registry → server, with migrations, rollbacks, env vars.</small>

## Data & AI extras

- [NumPy & pandas](/slides/data-ai/numpy-pandas.html)
  <br><small>Arrays, DataFrames, group-by, joining, time series.</small>
- [scikit-learn](/slides/data-ai/scikit-learn.html)
  <br><small>fit/predict, pipelines, cross-validation, model selection.</small>
- [LLM agents](/slides/llm-agents.html)
  <br><small>Anthropic / OpenAI APIs, tool use, the agent loop, RAG.</small>

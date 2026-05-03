---
title: Django templates
---

![Template with variable placeholders](/assets/images/topics/django-templates.svg)
<!-- .element: class="title-illustration" -->

# Django templates

Variables, tags, filters, blocks, forms.

---

## A first template

```django
{% raw %}<!-- templates/blog/detail.html -->
<!DOCTYPE html>
<html>
<head><title>{{ post.title }}</title></head>
<body>
  <h1>{{ post.title }}</h1>
  <p>{{ post.body }}</p>
  <p>By <strong>{{ post.author.name }}</strong></p>
</body>
</html>{% endraw %}
```

Two kinds of brackets:

- `{% raw %}{{ ... }}{% endraw %}` — output a variable
- `{% raw %}{% ... %}{% endraw %}` — a tag (control flow, includes, ...)

---

## Where templates live

Convention: each app has its own `templates/<app>/` directory.

```
blog/
├── views.py
└── templates/
    └── blog/                   ← namespaced!
        ├── list.html
        ├── detail.html
        └── form.html
```

The double `blog/` is intentional — prevents two apps from colliding on `list.html`.

```python
# In a view:
return render(request, "blog/detail.html", {"post": post})
```

---

## Variables and lookups

```django
{% raw %}{{ user }}              {# str(user) #}
{{ user.name }}         {# attribute or dict key #}
{{ items.0 }}           {# list index #}
{{ user.get_full_name }} {# method (called with no args) #}{% endraw %}
```

Django tries: dict key → attribute → list index → method (no parens). First match wins. **No method calls with arguments** in templates — that's by design (do it in the view).

---

## Filters

Transform a value before rendering:

```django
{% raw %}{{ name|lower }}                 {# 'alice' #}
{{ name|upper }}                 {# 'ALICE' #}
{{ name|default:"Anonymous" }}
{{ body|truncatewords:30 }}
{{ post.created_at|date:"Y-m-d" }}    {# 2026-05-01 #}
{{ count|pluralize }}            {# 's' if count != 1 #}
{{ items|length }}
{{ price|floatformat:2 }}        {# 9.99 #}
{{ html|safe }}                  {# don't escape #}
{{ text|linebreaks }}{% endraw %}
```

Pipe filters together:

```django
{% raw %}{{ post.body|truncatewords:50|linebreaks }}{% endraw %}
```

Built-in list at <https://docs.djangoproject.com/en/stable/ref/templates/builtins/>.

---

## Control flow

```django
{% raw %}{% if user.is_authenticated %}
  Welcome, {{ user.username }}.
{% elif show_signup %}
  <a href="{% url 'accounts:signup' %}">Sign up</a>
{% else %}
  <a href="{% url 'accounts:login' %}">Log in</a>
{% endif %}{% endraw %}
```

```django
{% raw %}{% for post in posts %}
  <li>{{ post.title }} ({{ forloop.counter }})</li>
{% empty %}
  <li>No posts yet.</li>
{% endfor %}{% endraw %}
```

`{% raw %}{% for ... %}{% empty %}{% endfor %}{% endraw %}` is the idiomatic "show empty state".

`forloop.counter`, `forloop.first`, `forloop.last`, `forloop.parentloop` are available.

---

## Template inheritance — base layout

Define a base layout, override sections in child templates.

```django
{% raw %}<!-- templates/base.html -->
<!DOCTYPE html>
<html>
<head><title>{% block title %}My site{% endblock %}</title></head>
<body>
  <header>...</header>
  <main>{% block content %}{% endblock %}</main>
  <footer>...</footer>
</body>
</html>{% endraw %}
```

`{% raw %}{% block name %}default{% endblock %}{% endraw %}` defines an override point.

--

## Template inheritance — child template

```django
{% raw %}<!-- templates/blog/detail.html -->
{% extends "base.html" %}

{% block title %}{{ post.title }}{% endblock %}

{% block content %}
  <h1>{{ post.title }}</h1>
  <p>{{ post.body }}</p>
{% endblock %}{% endraw %}
```

Child templates **fill blocks** from the parent. Anything outside `{% raw %}{% block %}{% endraw %}` in the child is ignored.

---

## `include` for reusable fragments

```django
{% raw %}<!-- templates/blog/_post_card.html -->
<article>
  <h2><a href="{% url 'blog:detail' post.id %}">{{ post.title }}</a></h2>
  <p>{{ post.body|truncatewords:30 }}</p>
</article>{% endraw %}
```

```django
{% raw %}<!-- templates/blog/list.html -->
{% for post in posts %}
  {% include "blog/_post_card.html" %}
{% endfor %}{% endraw %}
```

Use a leading underscore on partials by convention.

---

## URL reversal in templates

```django
{% raw %}<a href="{% url 'blog:detail' post.id %}">{{ post.title }}</a>
<a href="{% url 'blog:list' %}">All posts</a>
<form action="{% url 'blog:create' %}" method="post">
  {% csrf_token %}
  ...
</form>{% endraw %}
```

Always use `{% raw %}{% url %}{% endraw %}` — never hardcode paths.

---

## Static files

```django
{% raw %}{% load static %}
<link rel="stylesheet" href="{% static 'blog/style.css' %}">
<img src="{% static 'blog/logo.png' %}" alt="logo">{% endraw %}
```

Static files live in each app under `static/<app>/...`. In production, run `collectstatic` to copy them to a single directory your web server serves.

---

## Custom template tags and filters

In `blog/templatetags/blog_extras.py`:

```python
from django import template

register = template.Library()

@register.filter
def upto(text, max_words):
    return " ".join(text.split()[:max_words])

@register.simple_tag
def now_year():
    from datetime import datetime
    return datetime.now().year
```

Use:

```django
{% raw %}{% load blog_extras %}
<p>{{ post.body|upto:20 }}</p>
<footer>© {% now_year %}</footer>{% endraw %}
```

The `templatetags` directory needs an empty `__init__.py`.

---

## Forms — Django's `Form` class

```python
# blog/forms.py
from django import forms

class ContactForm(forms.Form):
    name = forms.CharField(max_length=100)
    email = forms.EmailField()
    message = forms.CharField(widget=forms.Textarea)
```

In a view:

```python
def contact(request):
    if request.method == "POST":
        form = ContactForm(request.POST)
        if form.is_valid():
            send_mail(**form.cleaned_data, from_email=...)
            return redirect("contact_done")
    else:
        form = ContactForm()
    return render(request, "blog/contact.html", {"form": form})
```

--

## Forms — rendering in templates

```django
{% raw %}<form method="post">
  {% csrf_token %}
  {{ form.as_p }}            {# each field wrapped in <p> #}
  <button type="submit">Send</button>
</form>{% endraw %}
```

Or render fields manually for full control:

```django
{% raw %}<form method="post">
  {% csrf_token %}
  <label>{{ form.name.label_tag }} {{ form.name }}</label>
  {{ form.name.errors }}
  <label>{{ form.email.label_tag }} {{ form.email }}</label>
  {{ form.email.errors }}
  <button type="submit">Send</button>
</form>{% endraw %}
```

`form.as_p` / `form.as_table` / `form.as_div` work for quick prototypes.

---

## ModelForms — half the boilerplate

```python
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "body", "tags"]
        widgets = {
            "body": forms.Textarea(attrs={"rows": 10}),
        }
```

`form.save()` writes to the DB:

```python
if form.is_valid():
    post = form.save(commit=False)   # don't write yet
    post.author = request.user        # add data not in the form
    post.save()
    form.save_m2m()                   # required for ModelForms with m2m fields
```

ModelForms cover 90% of CRUD form needs.

---

## Auto-escaping (XSS protection)

Django auto-escapes output by default:

```django
{% raw %}{{ comment.body }}      {# <script>alert(1)</script> → &lt;script&gt;... #}{% endraw %}
```

To opt out (for trusted HTML):

```django
{% raw %}{{ post.rendered_body|safe }}
{% autoescape off %}
  {{ trusted_html }}
{% endautoescape %}{% endraw %}
```

**Default to escaped.** Only mark `|safe` content you're sure of.

---

## Beyond DTL — alternatives

| Engine | When |
| --- | --- |
| **DTL** (built-in) | Default; simple; works with the admin and forms |
| **Jinja2** | More expressive (function calls with args, complex expressions) |
| **JS frontend** | When templates aren't enough — DRF for the API + React/Vue/HTMX |

For most Django apps, DTL is enough — and the rest of the framework integrates with it.

---

## What's next

- **Auth** — login, logout, session-based authentication
- **Authorization** — permissions, groups, decorators / mixins
- **DRF** — when you need JSON, not HTML

---
title: Django auth
---

![Lock and key for authentication](/assets/images/topics/django-auth.svg)
<!-- .element: class="title-illustration" -->

# Django authentication

Users, sessions, login, logout, password handling.

---

## What ships in `django.contrib.auth`

- A `User` model (username, email, password hash, flags)
- Password hashing (Argon2 / PBKDF2 / bcrypt)
- Session-based login
- Login / logout / password-change views and templates
- `@login_required` and `LoginRequiredMixin`
- Permissions and groups (we'll cover those in **Authorization**)

It's already in `INSTALLED_APPS` for new projects.

---

## The `User` model

```python
from django.contrib.auth import get_user_model

User = get_user_model()       # always use this — never `from auth.models import User`

User.objects.create_user(
    username="alice",
    email="a@example.com",
    password="secret",
)
# password is hashed automatically — never stored in plain text
```

`get_user_model()` returns whichever model `AUTH_USER_MODEL` points at — so you can swap to a custom user without touching the rest of the app.

---

## Authenticating a user

```python
from django.contrib.auth import authenticate, login, logout

def login_view(request):
    if request.method == "POST":
        user = authenticate(
            request,
            username=request.POST["username"],
            password=request.POST["password"],
        )
        if user is not None:
            login(request, user)             # → starts a session
            return redirect("home")
        return render(request, "login.html", {"error": "bad credentials"})
    return render(request, "login.html")

def logout_view(request):
    logout(request)
    return redirect("home")
```

`authenticate` returns the `User` on success, `None` on failure.

---

## Shortcut — built-in views

`django.contrib.auth.urls` ships login/logout/password-change/password-reset views.

```python
# mysite/urls.py
urlpatterns = [
    path("accounts/", include("django.contrib.auth.urls")),
]
```

Provides:

- `accounts/login/` — login form
- `accounts/logout/`
- `accounts/password_change/`
- `accounts/password_reset/` and the email flow

You only need to write the templates (`registration/login.html`, etc.).

---

## `@login_required` (FBV)

```python
from django.contrib.auth.decorators import login_required

@login_required
def dashboard(request):
    return render(request, "dashboard.html", {"user": request.user})
```

Anonymous users get redirected to `LOGIN_URL` (default `/accounts/login/`).

---

## `LoginRequiredMixin` (CBV)

```python
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import TemplateView

class Dashboard(LoginRequiredMixin, TemplateView):
    template_name = "dashboard.html"
    login_url = "/accounts/login/"        # optional override
```

Mixin goes **before** the base view class in the MRO.

---

## `request.user` — what to expect

```python
def my_view(request):
    request.user.is_authenticated    # True / False
    request.user.is_anonymous        # opposite
    request.user.is_staff            # admin-area access
    request.user.is_superuser        # all permissions

    request.user.username            # 'alice'
    request.user.email
    request.user.last_login
```

Anonymous users get an `AnonymousUser` instance — same interface, `is_authenticated == False`.

---

## In templates

```django
{% raw %}{% if user.is_authenticated %}
  <p>Welcome, {{ user.username }}.</p>
  <form method="post" action="{% url 'logout' %}">
    {% csrf_token %}
    <button type="submit">Log out</button>
  </form>
{% else %}
  <a href="{% url 'login' %}">Log in</a>
{% endif %}{% endraw %}
```

`user` is auto-injected via the `auth` context processor.

---

## Custom user model

If you want email-as-login, extra fields, or any meaningful customization — define your own user model **at project start**:

```python
# accounts/models.py
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    email = models.EmailField(unique=True)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to="avatars/", blank=True)
```

```python
# settings.py
AUTH_USER_MODEL = "accounts.User"
```

Switching `AUTH_USER_MODEL` after data exists is **painful** — set it at the very first migration.

---

## Password hashing

Django hashes passwords with PBKDF2 by default. Add Argon2 for stronger hashing:

```bash
uv add argon2-cffi
```

```python
# settings.py
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",     # legacy passwords
    "django.contrib.auth.hashers.PBKDF2SHA1PasswordHasher",
    "django.contrib.auth.hashers.BCryptSHA256PasswordHasher",
]
```

Old passwords are upgraded to the first hasher on next login. **Never** roll your own hasher.

---

## Password validators

```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {"NAME": "django.contrib.auth.password_validation.UserAttributeSimilarityValidator"},
    {"NAME": "django.contrib.auth.password_validation.MinimumLengthValidator",
     "OPTIONS": {"min_length": 12}},
    {"NAME": "django.contrib.auth.password_validation.CommonPasswordValidator"},
    {"NAME": "django.contrib.auth.password_validation.NumericPasswordValidator"},
]
```

Run on `set_password()` and registration forms. Add custom validators for company-specific rules.

---

## Sessions

After `login()`, Django stores the user's ID in a server-side session, keyed by a cookie.

```python
# settings.py
SESSION_COOKIE_SECURE = True        # HTTPS only
SESSION_COOKIE_HTTPONLY = True      # JS can't read it
SESSION_COOKIE_SAMESITE = "Lax"
SESSION_COOKIE_AGE = 60 * 60 * 24 * 14   # 2 weeks
```

Default backend stores session data in the DB (`django_session` table). For high-traffic sites, switch to Redis: `SESSION_ENGINE = "django.contrib.sessions.backends.cache"`.

---

## CSRF protection

Django's CSRF middleware blocks state-changing requests without a valid token.

```django
{% raw %}<form method="post">
  {% csrf_token %}
  ...
</form>{% endraw %}
```

For AJAX, send the cookie value as the `X-CSRFToken` header. Don't disable CSRF unless you have a different defense (typed APIs with tokens — covered in DRF).

---

## django-allauth — when built-ins aren't enough

For social login (Google, GitHub, ...), email-as-username, MFA, signup workflow:

```bash
uv add "django-allauth[socialaccount]"
```

```python
# settings.py
INSTALLED_APPS = [
    ...,
    "django.contrib.sites",
    "allauth",
    "allauth.account",
    "allauth.socialaccount",
    "allauth.socialaccount.providers.google",
]
SITE_ID = 1
```

Adds `accounts/` URLs with signup, login, social, password-reset — all wired up.

---

## Testing auth

```python
# tests/test_auth.py
from django.contrib.auth import get_user_model
from django.test import Client

User = get_user_model()

def test_dashboard_requires_login(db):
    response = Client().get("/dashboard/")
    assert response.status_code == 302       # redirect to login

def test_dashboard_for_logged_in(db):
    User.objects.create_user(username="alice", password="pw")
    c = Client()
    c.login(username="alice", password="pw")
    assert c.get("/dashboard/").status_code == 200
```

`Client` has a `.login()` shortcut that bypasses the form and sets the session.

---

## What's next

- **Authorization** — what a logged-in user can *do*
- **DRF** — token / JWT auth for APIs

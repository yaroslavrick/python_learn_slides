---
title: Refactoring Django
---

![Layered architecture: View, Service, Model, Database](/assets/images/topics/refactoring-django.svg)
<!-- .element: class="title-illustration" -->

# Refactoring Django

Fat models, thin views, services for the rest.

---

## The classic Django smell

The "everything in views" anti-pattern:

```python
def create_order(request):
    if request.method != "POST":
        return render(request, "form.html")
    cart = Cart.objects.get(user=request.user)
    if cart.is_empty():
        return JsonResponse({"error": "empty cart"}, status=400)
    total = sum(i.price * i.qty for i in cart.items.all())
    if total > request.user.credit_limit:
        return JsonResponse({"error": "over limit"}, status=402)
    order = Order.objects.create(user=request.user, total=total)
    for item in cart.items.all():
        OrderItem.objects.create(order=order, product=item.product, ...)
    cart.items.all().delete()
    send_confirmation_email.delay(order.id)
    log_audit(request.user, "order.created", order.id)
    return JsonResponse({"id": order.id})
```

---

## What's wrong with it

- **Hard to test** — you need a request, a session, a logged-in user
- **Hard to reuse** — calling this from a CLI command needs a fake request
- **Hard to read** — HTTP, business logic, persistence, side effects all mixed
- **Hard to change** — adding "place order" via the API duplicates everything

The view should answer **"what HTTP do I produce?"** — not **"how does an order get placed?"**

---

## Fat models, thin views — model

Move row-level logic onto the model.

```python
class Order(models.Model):
    @classmethod
    def from_cart(cls, cart):
        if cart.is_empty():
            raise EmptyCart()
        total = cart.total()
        if total > cart.user.credit_limit:
            raise CreditLimitExceeded(total)
        order = cls.objects.create(user=cart.user, total=total)
        for item in cart.items.all():
            OrderItem.objects.create(order=order, product=item.product, ...)
        cart.items.all().delete()
        return order
```

The model now owns the "an order can be created from a cart" rule.

--

## Fat models, thin views — view

The view becomes just HTTP plumbing:

```python
def create_order(request):
    if request.method != "POST":
        return render(request, "form.html")
    try:
        order = Order.from_cart(request.user.cart)
    except EmptyCart:
        return JsonResponse({"error": "empty cart"}, status=400)
    except CreditLimitExceeded:
        return JsonResponse({"error": "over limit"}, status=402)
    return JsonResponse({"id": order.id})
```

Same business rules now testable without a `request`, and reusable from a CLI command or task.

---

## When models get too fat — services

If `Order.from_cart` does email, logging, payment, analytics — the model is **doing too much**. Extract a service: a plain function that orchestrates models and side effects.

```python
# orders/services.py
from dataclasses import dataclass

@dataclass
class CreateOrderResult:
    order: Order

def create_order(user) -> CreateOrderResult:
    cart = user.cart
    if cart.is_empty():
        raise EmptyCart()
    if cart.total() > user.credit_limit:
        raise CreditLimitExceeded(cart.total())
    # ...continued...
```

--

## The service body

```python
def create_order(user) -> CreateOrderResult:
    # ...validation as before...

    with transaction.atomic():
        order = Order.objects.create(user=user, total=cart.total())
        for item in cart.items.all():
            OrderItem.objects.create(order=order, product=item.product, ...)
        cart.items.all().delete()

    send_confirmation_email.delay(order.id)
    audit.log(user, "order.created", order.id)
    return CreateOrderResult(order=order)
```

DB writes inside `transaction.atomic()`; side effects (email, audit) **after** the commit so a rollback doesn't fire them.

--

## Services — calling site

```python
# orders/views.py
from orders import services

def create_order_view(request):
    if request.method != "POST":
        return render(request, "form.html")
    try:
        result = services.create_order(request.user)
    except (EmptyCart, CreditLimitExceeded) as e:
        return JsonResponse({"error": str(e)}, status=e.http_status)
    return JsonResponse({"id": result.order.id})
```

Same service is callable from:

- HTTP views (above)
- DRF viewsets
- `manage.py` commands
- Celery tasks
- Tests (no request, no client)

---

## Custom QuerySets — defining one

Move `Post.objects.filter(published=True, ...)` into the model.

```python
class PostQuerySet(models.QuerySet):
    def published(self):
        return self.filter(published=True)

    def by_author(self, user):
        return self.filter(author=user)

    def recent(self, days=7):
        cutoff = timezone.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff)

class Post(models.Model):
    ...
    objects = PostQuerySet.as_manager()
```

---

## Custom QuerySets — using them

```python
Post.objects.published()                          # all published
Post.objects.published().by_author(alice).recent(30)
Post.objects.recent().count()
```

Custom QuerySets are **chainable**. Custom managers are not.

---

## Replace request-coupled code with pure functions

```python
# Bad: needs a request
def discount_for(request, order):
    if request.user.is_premium and order.total > 100:
        return Decimal("0.20")
    return Decimal("0")
```

```python
# Better: takes only what it needs
def discount_for(order: Order) -> Decimal:
    if order.user.is_premium and order.total > 100:
        return Decimal("0.20")
    return Decimal("0")
```

Pure functions are trivially testable — no fixtures, no mocks. Drop the `request` whenever you can.

---

## Forms as the validation layer

Don't validate POST data manually in views:

```python
# Bad — validation tangled with HTTP plumbing
def create_post(request):
    title = request.POST.get("title", "")
    if len(title) < 5:
        return HttpResponseBadRequest("title too short")
    ...
```

--

## Forms — the better version

```python
class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ["title", "body"]

    def clean_title(self):
        title = self.cleaned_data["title"]
        if len(title) < 5:
            raise ValidationError("title too short")
        return title

def create_post(request):
    form = PostForm(request.POST or None)
    if form.is_valid():
        return redirect(form.save())
    return render(request, "form.html", {"form": form})
```

Validation lives **with the data shape**, not in views.

---

## Don't use signals for business logic

Signals are great for **truly independent side effects** (cache invalidation, search indexing). They're poor for sequential business steps:

```python
# Hard to follow:
post_save.connect(create_invoice, sender=Order)
post_save.connect(send_email,     sender=Order)
post_save.connect(update_stock,   sender=Order)
post_save.connect(log_audit,      sender=Order)
```

You can't see in the view that all four things happen. Refactor to a service:

```python
def place_order(...):
    order = Order.objects.create(...)
    create_invoice(order)
    send_email(order)
    update_stock(order)
    log_audit(order)
    return order
```

Explicit > implicit when steps **depend on each other**.

---

## Avoid catching `Exception`

```python
# Bad
try:
    order = create_order(...)
except Exception:
    return JsonResponse({"error": "something went wrong"}, status=500)
```

You've swallowed `KeyboardInterrupt`, `MemoryError`, programming bugs. Prefer **typed domain exceptions**:

```python
class OrderError(Exception): ...
class EmptyCart(OrderError): http_status = 400
class CreditLimitExceeded(OrderError): http_status = 402
class StockExhausted(OrderError): http_status = 409

try:
    create_order(...)
except OrderError as e:
    return JsonResponse({"error": str(e)}, status=e.http_status)
```

Now `Exception` propagates → 500 + Sentry alert. Domain errors propagate → meaningful HTTP.

---

## Use type hints in services

```python
from decimal import Decimal

def create_order(user: User, cart: Cart) -> Order:
    if cart.is_empty():
        raise EmptyCart()
    return Order.objects.create(user=user, total=cart.total())

def discount_for(order: Order) -> Decimal:
    ...
```

Type hints make the contract visible at every call site. Run `mypy` in CI to keep them honest.

---

## Tests, not request integration tests

Prefer testing services directly:

```python
def test_create_order_empty_cart_raises(db):
    user = UserFactory()
    cart = CartFactory(user=user)
    with pytest.raises(EmptyCart):
        services.create_order(user)
```

vs

```python
def test_create_order_view_with_empty_cart(db):
    c = Client()
    user = UserFactory()
    c.force_login(user)
    response = c.post("/orders/")
    assert response.status_code == 400
```

Both are useful, but the service-level test catches the same bug in 10× less code. Keep the view-level test for the HTTP wiring.

---

## A practical layering

```
URL ─→ View (HTTP only)
         │
         ↓ calls
       Service (business logic)
         │
         ↓ uses
       Model + QuerySet (data access)
         │
         ↓ talks to
       Database
```

Each layer is testable in isolation; each has one reason to change.

---

## What's next

- **BDD** — testing through user-story scenarios
- **DRF** — services as the layer your API serializers call
- **Celery** — services scheduled in the background

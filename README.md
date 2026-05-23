# Mini-Blog
A simple blog project for practice.


# Implements

* Django REST Framework
* API Throttling
* Implement Radis Cache Management



Absolutely. I’ll teach you DRF throttling step-by-step using the official docs as the foundation, but in a more beginner-friendly and practical way.

Official reference: [DRF Throttling Docs](https://www.django-rest-framework.org/api-guide/throttling/?utm_source=chatgpt.com)

---

# What is throttling?

Throttling = limiting how many requests a client can make in a time period.

Examples:

* 100 requests/day
* 10 requests/minute
* 3 requests/second

If the user exceeds the limit:

```http
HTTP 429 Too Many Requests
```

DRF raises:

```python
rest_framework.exceptions.Throttled
```

---

# Why use throttling?

Main reasons:

| Use case                    | Example                 |
| --------------------------- | ----------------------- |
| Prevent abuse               | Spam API requests       |
| Protect expensive endpoints | AI/image generation     |
| Fair usage                  | Free vs paid users      |
| Prevent accidental overload | Infinite frontend loops |

Important:
DRF throttling is **NOT DDoS protection**. Even the official docs warn about this. ([Django REST Framework][1])

Use:

* DRF throttling → app-level control
* NGINX/Cloudflare → real security layer

---

# DRF throttle types

DRF gives 3 main throttle classes:

| Throttle             | Limits                |
| -------------------- | --------------------- |
| `AnonRateThrottle`   | Anonymous users       |
| `UserRateThrottle`   | Logged-in users       |
| `ScopedRateThrottle` | Specific API sections |

We’ll learn them one by one.

---

# 1. Global throttling (most common)

This is the easiest starting point.

## Step 1 — Configure settings.py

```python id="m8v1"
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],

    "DEFAULT_THROTTLE_RATES": {
        "anon": "5/min",
        "user": "20/min",
    }
}
```

Meaning:

| User type | Allowed            |
| --------- | ------------------ |
| Anonymous | 5 requests/minute  |
| Logged in | 20 requests/minute |

---

# Step 2 — Create a test API

```python id="n2j4"
from rest_framework.views import APIView
from rest_framework.response import Response

class HelloView(APIView):

    def get(self, request):
        return Response({"message": "Hello"})
```

---

# Step 3 — Test it

Call endpoint repeatedly:

```bash id="l2k8"
curl http://127.0.0.1:8000/api/hello/
```

After limit exceeded:

```json
{
    "detail": "Request was throttled."
}
```

---

# How DRF identifies users

## `AnonRateThrottle`

Uses:

* IP address

Example:

* 3 people behind same WiFi → same IP → shared limit

---

## `UserRateThrottle`

Uses:

* authenticated user ID

Example:

* user 15 gets independent limit
* user 16 gets separate limit

If not authenticated:

* falls back to IP address

Official behavior from DRF docs. ([Django REST Framework][1])

---

# Understanding throttle rates

Format:

```python id="a1"
"number/period"
```

Examples:

```python id="a2"
"10/min"
"100/day"
"5/hour"
"2/sec"
```

Supported periods:

* `s`
* `m`
* `h`
* `d`

---

# 2. Per-view throttling

Sometimes:

* login endpoint → strict
* public API → relaxed

You can throttle specific views.

---

## Example

```python id="p9w2"
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle

class MyView(APIView):

    throttle_classes = [UserRateThrottle]

    def get(self, request):
        return Response({"ok": True})
```

This view now uses only:

* `UserRateThrottle`

Official pattern from docs. ([Django REST Framework][1])

---

# 3. Custom throttle classes

This is where DRF becomes powerful.

Suppose:

* login API → 5/minute
* expensive AI API → 2/minute

Create custom throttles.

---

## Example

```python id="x3m1"
from rest_framework.throttling import UserRateThrottle

class LoginThrottle(UserRateThrottle):
    scope = "login"
```

Now settings:

```python id="x3m2"
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_RATES": {
        "login": "5/min",
    }
}
```

Apply it:

```python id="x3m3"
class LoginView(APIView):
    throttle_classes = [LoginThrottle]

    def post(self, request):
        ...
```

---

# How `scope` works

The scope connects:

```python id="x4"
scope = "login"
```

to:

```python id="x5"
"login": "5/min"
```

Think of it like a lookup key.

---

# 4. Burst + sustained throttling (VERY important)

This is a production-level pattern.

You want:

* stop spam bursts
* also limit daily abuse

Example:

* max 10/minute
* max 1000/day

DRF supports multiple throttles together.

---

## Create two throttles

```python id="b1"
from rest_framework.throttling import UserRateThrottle

class BurstThrottle(UserRateThrottle):
    scope = "burst"

class SustainedThrottle(UserRateThrottle):
    scope = "sustained"
```

---

## Configure settings

```python id="b2"
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "api.throttles.BurstThrottle",
        "api.throttles.SustainedThrottle",
    ],

    "DEFAULT_THROTTLE_RATES": {
        "burst": "10/min",
        "sustained": "1000/day",
    }
}
```

Official example pattern from DRF docs. ([Django REST Framework][1])

---

# 5. ScopedRateThrottle

This is useful for API sections.

Example:

* uploads → expensive
* normal endpoints → cheap

---

## Example

```python id="s1"
from rest_framework.throttling import ScopedRateThrottle

class UploadView(APIView):

    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "uploads"

    def post(self, request):
        ...
```

Settings:

```python id="s2"
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.ScopedRateThrottle"
    ],

    "DEFAULT_THROTTLE_RATES": {
        "uploads": "3/min"
    }
}
```

Meaning:

* all upload endpoints share same quota

---

# 6. Function-based views

You can throttle FBVs too.

```python id="f1"
from rest_framework.decorators import api_view, throttle_classes
from rest_framework.throttling import UserRateThrottle
from rest_framework.response import Response

@api_view(["GET"])
@throttle_classes([UserRateThrottle])
def test_view(request):
    return Response({"hello": "world"})
```

Official docs example pattern. ([Django REST Framework][1])

---

# 7. Custom throttle logic

Sometimes built-in logic is not enough.

Example:

* randomly block traffic
* custom business rules
* API credits

---

## Simple custom throttle

```python id="c1"
import random
from rest_framework.throttling import BaseThrottle

class RandomThrottle(BaseThrottle):

    def allow_request(self, request, view):

        return random.randint(1, 5) != 1
```

If returns:

* `True` → allow
* `False` → block

Based on official docs example. ([Django REST Framework][1])

---

# 8. Where throttling data is stored

DRF uses Django cache.

Usually:

* Redis
* Memcached
* Local memory

Production recommendation:
Use Redis.

---

## BAD for production

```python id="cache1"
LocMemCache
```

Why?
Each process has separate memory.

Real-world issue discussed by Django users. ([Reddit][2])

---

# Production Redis setup

```python id="redis1"
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
    }
}
```

This is the common production setup.

---

# Real-world throttle strategy

Here’s a practical setup:

| Endpoint       | Limit   |
| -------------- | ------- |
| Login          | 5/min   |
| OTP verify     | 3/min   |
| Password reset | 2/min   |
| Normal API     | 100/min |
| AI endpoint    | 10/min  |

---

# Common beginner mistakes

## Mistake 1 — Using LocMemCache in production

Causes inconsistent throttling.

---

## Mistake 2 — Forgetting authentication

`UserRateThrottle` depends on authenticated users.

Without auth:

* falls back to IP.

---

## Mistake 3 — Thinking throttling = security

It is not full DDoS protection. ([Django REST Framework][1])

---

# Recommended learning path

Learn in this order:

1. Global throttles
2. Per-view throttles
3. Custom throttle classes
4. Scoped throttles
5. Redis cache
6. Custom `BaseThrottle`

That’s enough for most production APIs.

---

# Mini practice project

Try building:

## Blog API

Rules:

* anonymous → 10/min
* authenticated → 100/min
* comment endpoint → 5/min
* image upload → 2/min

This will teach almost everything.

---

# Most important takeaway

Think of throttling as:

```text
Who is making requests?
+
How many requests?
+
During what time?
+
For which endpoint?
```

That mental model makes DRF throttling much easier.

Docs used:

* [Official DRF Throttling Guide](https://www.django-rest-framework.org/api-guide/throttling/?utm_source=chatgpt.com)
* [DRF Throttling Tutorial Article](https://unwiredlearning.com/blog/drf-throttling-guide?utm_source=chatgpt.com)

[1]: https://www.django-rest-framework.org/api-guide/throttling/?utm_source=chatgpt.com "Throttling - Django REST framework"
[2]: https://www.reddit.com/r/django/comments/1hqgxm5?utm_source=chatgpt.com "Throttling works on local but does not on server. ADVICE NEEDED!"

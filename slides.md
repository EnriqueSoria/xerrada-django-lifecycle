[comment]: # (width: 1600)
[comment]: # (height: 900)
[comment]: # (THEME = white)
[comment]: # (CODE_THEME = routeros)


Enrique Soria

# django-lifecycle

Descobrint com gestionar amb Django el cicle de vida d'un model, _sense pels ni senyals_

[comment]: # (!!!)

### El nostre projecte

Suposem que volem crear un blog.

[comment]: # (!!! data-auto-animate)

### El nostre projecte

Suposem que volem crear un blog.

El nostre blog té publicacions.

[comment]: # (!!! )

### El nostre projecte

```python [4-7]
from django.db import models
from django.urls import reverse


class Post(models.Model):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def get_url(self, slug=None):
        slug = slug or self.slug
        return reverse("posts", kwargs={"slug": slug})
```

[comment]: # (!!!)

### Objectiu


Setejar l'_slug_ del nostre blog automàticament

¿Com ho farieu?

Note:
(no, no val a dir amb lifecycle)
 - Sobreescrivint el save?
 - Señales?

[comment]: # (!!!)

#### Sobreescrivint el `.save()` 

```python [2, 10-12]
from django.db import models
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def save(self, *args, **kwargs):
        self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```


[comment]: # (!!! data-auto-animate)

#### Sobreescrivint el `.save()`

...només quan creem la instància.

```python [11-12]
from django.db import models
from django.utils.text import slugify


class Post(models.Model):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def save(self, *args, **kwargs):
        if self.pk is None:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```



[comment]: # (!!!)

#### Sobreescrivint el `.save()`

No té molt de misteri. Però al cap d'un temps descobrim que Django té una cosa anomenada... 
[`signals`](https://docs.djangoproject.com/en/5.0/topics/signals/).

[comment]: # (!!!)

#### Amb senyals de Django

```python
# blog/signals.py
from django.core.signals import pre_save
from django.utils.text import slugify
from .models import Blog


@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    if instance.pk is None:
        instance.slug = slugify(instance.title)
```

Note:
Pinta bé, PERÒ s'ha de colocar fora del model

...i a més a més ens hem d'enrecordar de registrar la senyal al `app.ready`

[comment]: # (!!! data-auto-animate)

#### Amb senyals de Django

```python
# blog/signals.py
from django.core.signals import pre_save
from django.utils.text import slugify
from .models import Blog


@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    if instance.pk is None:
        instance.slug = slugify(instance.title)
```

```python
# blog/app.py
from django.apps import AppConfig


class MyAppConfig(AppConfig):
    def ready(self):
        from . import signals
```

[comment]: # (!!!)

Genial, ja hem aprés a setejar l'slug en el moment de la creació d'una publicació, però...

[comment]: # (!!!)

![la gent canvia](https://pbs.twimg.com/media/BmQIKItCUAA_yQP.jpg)

...i els tituls també

[comment]: # (!!!)

# Nous requeriments

 - Canviar l'slug quan el titul canvia
 - Fer un redirect de l'antiga URL a la nova

[comment]: # (!!!)

### Sobreescrivint el `.save()`

Canviar l'slug quan el titul canvie

```python [5, 8]
class Post(models.Model):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._original_title = self.title

    def save(self, *args, **kwargs):
        if self.pk is None or self._original_title != self.title:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

[comment]: # (!!!)

### Sobreescrivint el `.save()`

Fer un redirect de l'URL antiga a la nova

```python [12-13]
class Post(models.Model):
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._original_title = self.title
        self._original_slug = self.slug

    def save(self, *args, **kwargs):
        if self.pk is None or self._original_title != self.title:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
        if self._original_slug != self.slug:
            self.create_redirect(old_url=self.get_url(self._original_slug), new_url=self.get_url(self.slug))
```

[comment]: # (!!!)

### Amb senyals de Django

Canviar l'slug quan el titul canvie

```python [4, 6]
# blog/signals.py
@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    old_instance = Post.objects.get(pk=instance.pk)
    
    if instance.pk is None or old_instance.title != instance.title:
        instance.slug = slugify(instance.title)
```

[comment]: # (!!! data-auto-animate)

### Amb senyals de Django

Canviar l'slug quan el titul canvie

Fer un redirect de l'URL antiga a la nova

```python [7-13]
# blog/signals.py
@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    old_instance = Post.objects.get(pk=instance.pk)
    
    if instance.pk is None or old_instance.title != instance.title:
        instance.slug = slugify(instance.title)

    Post.create_redirect(self.create_redirect(
        old_instance=instance.get_url(old_instance.slug),
        new_url=instance.get_url(instance.slug))
    )
```


[comment]: # (!!!)

### Amb django-lifecycle

Hem vingut ací per a parlar de certa llibreria, veritat? 

[comment]: # (!!!)

### Amb django-lifecycle

Hem vingut ací per a parlar de certa llibreria, veritat? 

Anem a reescriure els exemples anteriors amb django-lifecycle:


```python [3, 6]
from django.db import models
from django.utils.text import slugify
from django_lifecycle import LifecycleModel


class Post(LifecycleModel):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()
```

[comment]: # (!!!)

### Amb django-lifecycle

Vamos a crear unas funciones que hagan lo que queremos.

```python
class Post(LifecycleModel):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def set_slug(self):
        self.slug = slugify(self.title)
```

[comment]: # (!!!)

### Amb django-lifecycle

`django-lifecycle` ens proporciona una utilitat per a obtindre el valor inicial: `initial_value(field_name)`

```python [9-14]
class Post(LifecycleModel):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def set_slug(self):
        self.slug = slugify(self.title)

    def create_redirect(self):
        Redirect.objects.update_or_create(
            site=1,
            old_path=self.get_absolute_url(slug=self.initial_value("slug")),
            new_path=self.get_absolute_url(slug=self.slug),
        )
```

[comment]: # (!!!)

### Amb django-lifecycle

Escollir [en quins moments](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#lifecycle-moments) volem realitzar estes accions.

[comment]: # (!!! data-auto-animate)

### Amb django-lifecycle

Escollir [en quins moments](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#lifecycle-moments) volem realitzar estes accions:

```
BEFORE_SAVE
AFTER_SAVE

BEFORE_CREATE
AFTER_CREATE

BEFORE_UPDATE
AFTER_UPDATE

BEFORE_DELETE
AFTER_DELETE
```

[comment]: # (!!!)

### Amb django-lifecycle

Escollir [en quins moments](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#lifecycle-moments) volem realitzar estes accions:

```python [3, 7]
class Post(LifecycleModel):

    @hook(BEFORE_SAVE)
    def set_slug(self):
        self.slug = slugify(self.title)

    @hook(AFTER_SAVE)
    def create_redirect(self):
        Redirect.objects.update_or_create(
            site=1,
            old_path=self.get_absolute_url(slug=self.initial_value("slug")),
            new_path=self.get_absolute_url(slug=self.slug),
        )
```

[comment]: # (!!!)

### Amb django-lifecycle

Encara que no sempre s'han d'executar. 

Volem especificar [quines condicions executen els hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

Note:
Requisits:
 - slugify BEFORE_CREATE i quan canvie el titul

[comment]: # (!!! data-auto-animate)

### Amb django-lifecycle


Encara que no sempre s'han d'executar. 

Volem especificar [quines condicions executen els hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

```python
WhenFieldHasChanged(field_name, has_changed)

WhenFieldValueIs(field_name, value)

WhenFieldValueIsNot(field_name, value)

WhenFieldValueWas(field_name, value)

WhenFieldValueWasNot(field_name, value)

WhenFieldValueChangesTo(field_name, value)
````

[comment]: # (!!!)

### Amb django-lifecycle

Encara que no sempre s'han d'executar. 

Volem especificar [quines condicions executen els hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

```python [3-4, 8]
class Post(LifecycleModel):

    @hook(BEFORE_CREATE)
    @hook(BEFORE_SAVE, condition=WhenFieldHasChanged("title"))
    def set_slug(self):
        self.slug = slugify(self.title)

    @hook(AFTER_SAVE, condition=WhenFieldHasChanged("slug"))
    def create_redirect(self):
        Redirect.objects.update_or_create(
            site=1,
            old_path=self.get_absolute_url(slug=self.initial_value("slug")),
            new_path=self.get_absolute_url(self.slug),
        )
```

[comment]: # (!!!)

### Amb django-lifecycle

Encara que no sempre s'han d'executar. 

Volem especificar [quines condicions executen els hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

```python [5]
class Post(LifecycleModel):

    @hook(
        BEFORE_SAVE,
        condition=WhenFieldHasChanged("title") | WhenFieldValueIs("slug", value=None)
    )
    def set_slug(self):
        self.slug = slugify(self.title)

    @hook(AFTER_SAVE, condition=WhenFieldHasChanged("slug"))
    def create_redirect(self):
        Redirect.objects.update_or_create(
            site=1,
            old_path=self.get_absolute_url(slug=self.initial_value("slug")),
            new_path=self.get_absolute_url(self.slug),
        )
```

[comment]: # (!!!)


### ¿Com podem asegurar que `set_slug` s'execute abans del `create_redirect`?

Amb el paràmetre `priority`:


[comment]: # (!!!)

### ¿Com podem asegurar que `set_slug` s'execute abans del `create_redirect`?

Amb el paràmetre `priority`:

[comment]: #  (!!! data-auto-animate)

```python [1, 10]
from django_lifecycle.priority import LOWER_PRIORITY

class Post(LifecycleModel):

    @hook(BEFORE_CREATE)
    @hook(BEFORE_SAVE, condition=WhenFieldHasChanged("title"))
    def set_slug(self):
        self.slug = slugify(self.title)

    @hook(AFTER_SAVE, condition=WhenFieldHasChanged("slug"), priority=LOWER_PRIORITY)
    def create_redirect(self):
        Redirect.objects.update_or_create(
            site=1,
            old_path=self.get_absolute_url(slug=self.initial_value("slug")),
            new_path=self.get_absolute_url(self.slug),
        )
```

Note: 
Esta va ser la meua primera PR a esta llibreria

[comment]: # (!!!)

# Nous requeriments

 - Impedir publicacions sobre IA al nostre blog

[comment]: # (!!!)

## Condicions personalitzades

Si les condicions que ens oferix django-lifecycle no ens apanyen, podem crear les nostres pròpies condicions:

```python
from django_lifecycle.conditions.base import ChainableCondition


class WhenFieldContains(ChainableCondition):
    def __init__(self, field_name, contains_value):
        self.field_name = field_name
        self.contains_value = contains_value
    
    def __call__(self, instance, update_fields=None):
        field_value = getattr(instance, self.field_name)
        return self.contains_value in field_value
```

[comment]: # (!!!)

## Condicions personalitzades

Si les condicions que ens oferix django-lifecycle no ens apanyen, podem crear les nostres pròpies condicions:

```python
class Post(LifecycleModel):
    @hook(
        BEFORE_SAVE, condition=WhenFieldContains("title", "IA") | WhenFieldContains("title", "LLM")
    )   
    def avoid_artificial_intelligence_posts(self):
        raise Exception("¡Los artículos sobre IA están prohibidos en esta casa!")
```


[comment]: # (!!!)

## ¿Quan elegir cada opció?
 - django signals
 - sobreescriure `.save()`
 - django-lifecycle

Note:
Ara que ja hem vist lo fantàstic que es django-lifecycle...

[comment]: # (!!!)

## ¿Quan elegir cada opció?
### Django signals

- Models d'apps de tercers
- Comportaments genèrics per a molts models
- Accions no relacionades amb el cicle de vida d'un model (i.e.: [django-allauth](https://docs.allauth.org/en/latest/account/signals.html))

Note: 
django-allauth:
 - user_logged_in(request, user)
 - password_set(request, user)
 - email_changed(request, user, from_email_address, to_email_address)


[comment]: # (!!!)

## ¿Quan elegir cada opció?
### Sobreescriure el `.save()`

- Casos d'ús molt simples

Note:
per no haver d'afegir una dependencia externa

[comment]: # (!!!)

## Para todo lo demás... 

[comment]: # (!!!)

## Para todo lo demás...

![master-card](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b7/MasterCard_Logo.svg/1920px-MasterCard_Logo.svg.png)

[comment]: # (!!!)

## Para todo lo demás... `django-lifecycle`

[comment]: # (!!!)

### Limitacions de `django-lifecycle`

Degut al funcionament de django-lifecycle (sobreescriu el `.save`), existeixen [certes limitacions](https://github.com/rsinger86/django-lifecycle/issues/143):

- No s'executa quan realitzes accions en `bulk`. (les senyals de Django si) 
  - Deletes a l'admin de Django
  - Deletes en cascada a l'eliminar un objecte relacionat
- Relacions many2many (s'executen al `through`)

[comment]: # (!!!)

## django-lifecicle
En conclusió: la llibreria django-lifecycle ens permeteix, de forma declarativa, crear una sèrie d'accions asociades
al cicle de vida d'un model de Django.

[comment]: # (!!!)

## Fin

[comment]: # (!!!)

# Enrique Soria

- Github: [@EnriqueSoria](https://github.com/EnriqueSoria)
- Twitter: [@esoria_dev](https://twitter.com/esoria_dev)
- Mastodon: [@esoria_dev@mastodon.social](https://mastodon.social/@esoria_dev)
- [Enrique Soria Magraner](https://www.linkedin.com/in/enriquesoriamagraner/) en Linkedin

[comment]: # (width: 1600)
[comment]: # (height: 900)
[comment]: # (THEME = white)
[comment]: # (CODE_THEME = routeros)


Enrique Soria

# django-lifecycle

Descubriendo cómo gestionar en Django el ciclo de vida de un modelo, _sin pelos ni señales_

[comment]: # (!!!)

### Nuestro proyecto

Vamos a suponer que estamos creando un blog.

[comment]: # (!!! data-auto-animate)
### Nuestro proyecto

Vamos a suponer que estamos creando un blog.

Nuestro blog tiene publicaciones.

[comment]: # (!!! )

### Nuestro proyecto

```python [4-7]
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def get_url(self, slug=None):
        slug = slug or self.slug
        return reverse("posts", kwargs={"slug": slug})
```

[comment]: # (!!!)

### Objetivo

Setear el _slug_ de nuestro flamante blog automáticamente.

¿Cómo lo haríais?

Note:
(no, no vale decir con lifecycle)
 - Sobreescribiendo el save?
 - Señales?

[comment]: # (!!!)

#### Sobreescribiendo el `.save()` 

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

#### Sobreescribiendo el `.save()`

...solamente cuando estamos creando la instancia

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

#### Sobreescribiendo el `.save()`

No tiene mucho misterio. Aunque un tiempo después nos enteramos que django tiene una cosa
llamada... [`signals`](https://docs.djangoproject.com/en/5.0/topics/signals/).

[comment]: # (!!!)

#### Usando señales de Django

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
Pinta bien, PERO esto hay que ubicarlo fuera del modelo

...y ADEMÁS nos tenemos que acordar de registrar la señal en el `app.ready`

[comment]: # (!!! data-auto-animate)

#### Usando señales de Django

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

¡Genial, ya hemos aprendido a setear el slug en el momento de creación de una publicación, pero...

[comment]: # (!!!)

![la gente cambia](https://pbs.twimg.com/media/BmQIKItCUAA_yQP.jpg)

...y los títulos también

[comment]: # (!!!)

# Nuevos requerimientos

 - Cambiar el slug cuando el título cambia
 - Hacer un redirect de la antigua URL a la nueva

[comment]: # (!!!)

### Sobreescribiendo el `.save()`

Cambiar el slug cuando el título cambia

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

### Sobreescribiendo el `.save()`

Hacer un redirect de la antigua URL a la nueva

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

### Usando señales de Django

Cambiar el slug cuando el título cambia

```python [4, 6]
# blog/signals.py
@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    old_instance = Post.objects.get(pk=instance.pk)
    
    if instance.pk is None or old_instance.title != instance.title:
        instance.slug = slugify(instance.title)
```

[comment]: # (!!! data-auto-animate)

### Usando señales de Django

Cambiar el slug cuando el título cambia

Hacer un redirect de la antigua URL a la nueva

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

### Usando django-lifecycle

Hemos venido a hablar de cierta librería, ¿verdad? 

Vamos a reescribir los antiguos ejemplos usando django-lifecycle:


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

### Usando señales de Django

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

### Usando señales de Django

`django-lifecycle` nos proporciona una utilidad para obtener el valor inicial: `initial_value(field_name)`

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

### Usando señales de Django

#### Escoger [en qué momentos](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#lifecycle-moments) se quieren realizar estas acciones.

[comment]: # (!!! data-auto-animate)

### Usando señales de Django

#### Escoger [en qué momentos](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#lifecycle-moments) se quieren realizar estas acciones:

 - BEFORE_SAVE, AFTER_SAVE
 - BEFORE_CREATE, AFTER_CREATE
 - BEFORE_UPDATE, AFTER_UPDATE
 - BEFORE_DELETE, AFTER_DELETE

[comment]: # (!!!)

### Usando señales de Django

####  Escoger [en qué momentos](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#lifecycle-moments) se quieren realizar estas acciones:

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

### Usando señales de Django

Aunque no siempre se tienen que ejecutar. 

Hay que especificar [en qué condiciones se ejecutan los hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

Note:
Requisitos:
 - slugify BEFORE_CREATE y cuando cambie el título

[comment]: # (!!! data-auto-animate)

### Usando señales de Django


Aunque no siempre se tienen que ejecutar. 

Hay que especificar [en qué condiciones se ejecutan los hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

```python
WhenFieldHasChanged(field_name, has_changed)

WhenFieldValueIs(field_name, value)

WhenFieldValueIsNot(field_name, value)

WhenFieldValueWas(field_name, value)

WhenFieldValueWasNot(field_name, value)

WhenFieldValueChangesTo(field_name, value)
````

[comment]: # (!!!)

### Usando señales de Django

Aunque no siempre se tienen que ejecutar. 

Hay que especificar [en qué condiciones se ejecutan los hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

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

### Usando señales de Django

Aunque no siempre se tienen que ejecutar. 

Hay que especificar [en qué condiciones se ejecutan los hooks](https://rsinger86.github.io/django-lifecycle/hooks_and_conditions/#conditions).

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


### ¿Cómo nos podemos asegurar que `set_slug` se ejecute antes que el `create_redirect`?

Con el parámetro `priority`:


[comment]: # (!!!)

### ¿Cómo nos podemos asegurar que `set_slug` se ejecute antes que el `create_redirect`?

Con el parámetro `priority`:

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
Esta fué mi primera PR a esta librería.

[comment]: # (!!!)

# Nuevos requerimientos

 - Impedir publicaciones sobre IA en nuestro blog

[comment]: # (!!!)

## Custom conditions

Si las condiciones que existen no nos convencen podemos crear nuestras propias condiciones:

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

Si las condiciones que existen no nos convencen podemos crear nuestras propias condiciones:

```python
class Post(LifecycleModel):
    @hook(
        BEFORE_SAVE, condition=WhenFieldContains("title", "IA") | WhenFieldContains("title", "LLM")
    )   
    def avoid_artificial_intelligence_posts(self):
        raise Exception("¡Los artículos sobre IA están prohibidos en esta casa!")
```


[comment]: # (!!!)

# ¿Cuándo elegir cada opción?
 - django signals
 - sobreescribir `.save()`
 - django-lifecycle

Note:
Ahora que hemos visto lo maravilloso que es django-lifecycle...

[comment]: # (!!!)

## Django signals

- Modelos de apps de terceros
- Comportamientos genéricos para muchos modelos
- Acciones no relacionadas con el ciclo de vida de un modelo (i.e.: [django-allauth](https://docs.allauth.org/en/latest/account/signals.html))

Note: 
django-allauth:
 - user_logged_in(request, user)
 - password_set(request, user)
 - email_changed(request, user, from_email_address, to_email_address)


[comment]: # (!!!)

## Sobreescribir el `.save()`

- Casos de usos muy simples

Note:
no añadir dependencia externa

[comment]: # (!!!)

## Para todo lo demás... 


[comment]: # (!!!)

## Para todo lo demás...

![master-card](https://upload.wikimedia.org/wikipedia/commons/thumb/b/b7/MasterCard_Logo.svg/1920px-MasterCard_Logo.svg.png)

[comment]: # (!!!)

## Para todo lo demás... `django-lifecycle`

[comment]: # (!!!)

### Limitaciones de `django-lifecycle`

Debido a cómo funciona (sobreescribiendo el `.save`), existen [ciertas limitaciones](https://github.com/rsinger86/django-lifecycle/issues/143):

- No se ejecuta cuando realizas acciones `bulk`. (las señales de Django si) 
  - Deletes en el admin de Django
  - Deletes en cascada al eliminar un objeto relacionado
- No se ejecutan con relaciones M2M (se ejecutan en el `through`)

[comment]: # (!!!)

En conclusión, la librería django-lifecycle nos permite, de forma declarativa, crear una serie de acciones asociadas al
ciclo de vida de un modelo Django.

[comment]: # (!!!)

## Fin

[comment]: # (!!!)

# Enrique Soria

- Github: [@EnriqueSoria](https://github.com/EnriqueSoria)
- Twitter: [@esoria_dev](https://twitter.com/esoria_dev)
- Mastodon: [@esoria_dev@mastodon.social](https://mastodon.social/@esoria_dev)
- [Enrique Soria Magraner](https://www.linkedin.com/in/enriquesoriamagraner/) en Linkedin

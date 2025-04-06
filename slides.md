[comment]: # (width: 1600)
[comment]: # (height: 900)
[comment]: # (THEME = white)
[comment]: # (CODE_THEME = routeros)


Enrique Soria


# django-lifecycle

Descobrint com gestionar en Django el cicle de vida d’un model, _sense pèls ni senyals_

### El nostre projecte

Suposem que estem creant un blog.

El nostre blog té publicacions.

```python
from django.db import models


class Post(models.Model):
    title = models.CharField(max_length=32)
    slug = models.SlugField(max_length=32)
    body = models.TextField()

    def get_url(self, slug=None):
        slug = slug or self.slug
        return reverse("posts", kwargs={"slug": slug})
```

### Objectiu

Establir el _slug_ del nostre flamant blog automàticament.

Com ho faríeu?

(no, no val dir amb lifecycle)
 - Sobreescrivint el `save`?
 - Senyals?

#### Sobreescrivint el `.save()` 

```python
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

#### Sobreescrivint el `.save()`

...només quan estem creant la instància

```python
    def save(self, *args, **kwargs):
        if self.pk is None:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

#### Sobreescrivint el `.save()`

No té gaire misteri. Tot i que, un temps després, ens assabentem que Django té una cosa anomenada [`signals`](https://docs.djangoproject.com/en/5.0/topics/signals/).

#### Usant senyals de Django

```python
@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    if instance.pk is None:
        instance.slug = slugify(instance.title)
```

*Però cal ubicar-ho fora del model i recordar registrar la senyal a `app.ready`*

```python
class MyAppConfig(AppConfig):
    def ready(self):
        from . import signals
```

---

### Nous requeriments

 - Canviar el slug quan el títol canviï
 - Fer un redirect de la URL antiga a la nova

#### Sobreescrivint el `.save()`

Canviar el slug quan el títol canvia:

```python
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._original_title = self.title

    def save(self, *args, **kwargs):
        if self.pk is None or self._original_title != self.title:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
```

Fer un redirect:

```python
    def save(self, *args, **kwargs):
        if self.pk is None or self._original_title != self.title:
            self.slug = slugify(self.title)
        super().save(*args, **kwargs)
        if self._original_slug != self.slug:
            self.create_redirect(old_url=self.get_url(self._original_slug), new_url=self.get_url(self.slug))
```

#### Amb senyals

```python
@receiver(pre_save, sender=Post)
def on_post_pre_save_receiver(sender, instance, raw, using, update_fields):
    old_instance = Post.objects.get(pk=instance.pk)
    
    if instance.pk is None or old_instance.title != instance.title:
        instance.slug = slugify(instance.title)
```

Amb redirect:

```python
    Post.create_redirect(
        old_instance=instance.get_url(old_instance.slug),
        new_url=instance.get_url(instance.slug)
    )
```

---

## Usant django-lifecycle

```python
from django_lifecycle import LifecycleModel, hook, BEFORE_SAVE, AFTER_SAVE
```

```python
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

També podem afegir condicions:

```python
@hook(BEFORE_CREATE)
@hook(BEFORE_SAVE, condition=WhenFieldHasChanged("title"))
```

---

## Nous requeriments

 - Impedir publicacions sobre IA

Condició personalitzada:

```python
class WhenFieldContains(ChainableCondition):
    def __call__(self, instance, update_fields=None):
        field_value = getattr(instance, self.field_name)
        return self.contains_value in field_value
```

```python
@hook(
    BEFORE_SAVE, condition=WhenFieldContains("title", "IA") | WhenFieldContains("title", "LLM")
)
def avoid_artificial_intelligence_posts(self):
    raise Exception("Els articles sobre IA estan prohibits en aquesta casa!")
```

---

## Quan triar cada opció?

### Django signals
- Models d'apps de tercers
- Comportaments genèrics per molts models
- Accions no relacionades amb el cicle de vida

### `.save()`
- Casos molt senzills
- Evitar dependències externes

### django-lifecycle
- Per tot el demés 🧡

Limitacions:
- No funciona amb operacions bulk
- No es dispara en relacions M2M (només via `through`)

---

## Fi 🎉

[comment]: # (!!!)

# Enrique Soria

- Github: [@EnriqueSoria](https://github.com/EnriqueSoria)
- Twitter: [@esoria_dev](https://twitter.com/esoria_dev)
- Mastodon: [@esoria_dev@mastodon.social](https://mastodon.social/@esoria_dev)
- [Enrique Soria Magraner](https://www.linkedin.com/in/enriquesoriamagraner/) en Linkedin

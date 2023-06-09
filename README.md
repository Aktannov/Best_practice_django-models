# Django Models Best Practices


## Содержание
1. [Правильное определение названия модели](#1-правильное-определение-названия-модели)
2. [Название полей связей(связанных объектов)](#2-название-полей-связейсвязанных-объектов)
3. [Правильное указание названия связанного объекта.](#3-правильное-указание-названия-связанного-объекта)
4. [Не используйте ForeignKey с unique=True](#4-не-используйте-foreignkey-с-uniquetrue)
5. [Порядок атрибутов и методов в модели](#5-порядок-атрибутов-и-методов-в-модели)
6. [Использование вариантов выбора (CHOICES)](#6-использование-вариантов-выбора-choices)
7. [Много флагов в модели?](#7-много-флагов-в-модели)
8. [Не складывайте все загруженные пользователем файлы в одну папку.](#8-не-складывайте-все-загруженные-пользователем-файлы-в-одну-папку)
9. [Используйте абстрактные модели](#9-используйте-абстрактные-модели)
10. [Использование пользовательского менеджера и QuerySet](#10-использование-пользовательского-менеджера-и-queryset)

#### 1. Правильное определение названия модели
Обычно рекомендуется использовать существительные единственного
числа для именования моделей, например:
* User
* Post
* Article

То есть,последний компонент названия должен быть существительным, например: __Some New Shiny Item__. Правильно использовать единственное число, когда одна единица модели не содержит информации о нескольких объектах.

#### 2. Название полей связей(связанных объектов) 
Для таких связей, как __ForeignKey__, __OneToOneKey__, __ManyToMany__, иногда лучше указать имя. Представьте, что есть модель __Article__, - в которой одним из связей является __ForeignKey__ для модели __User__. Если это поле содержит информацию об авторе статьи, то __author__ будет более подходящим именем, чем __user__.


#### 3. Правильное указание названия связанного объекта. 
Целесообразно указывать связанное имя во множественном числе, так как обращение к связанному имени возвращает __queryset__. Пожалуйста, задавайте адекватные связанные имена. В большинстве случаев имя модели во множественном числе будет правильным.

Например:

```python
class Owner(models.Model):
    pass
class Item(models.Model):
    owner = models.ForeignKey(Owner, related_name='items')
```


#### 4. Не используйте ForeignKey с unique=True 
Нет смысла использовать __ForeignKey__ с __unique=True__, поскольку для таких
случаев существует __OneToOneField__.



#### 5. Порядок атрибутов и методов в модели 
Предпочтительный порядок атрибутов и методов в модели.
* константы (для выбора и другие)
* поля модели
* указание пользовательского менеджера
* __meta__
* __def__ __\__unicode____ __(python 2)__ или __def__ __\__str____ __(python 3)__
* другие специальные методы
* __def clean__
* __def save__
* __def get_absolut_url__
* другие методы
Обратите внимание, что приведенный порядок был взят из документации и
немного расширен.


#### 6. Использование вариантов выбора (CHOICES) 
При использовании вариантов выбора рекомендуется:

* хранить в базе данных строки вместо чисел (хотя это не лучший вариант
с точки зрения использования необязательной базы данных, на
практике он удобнее, так как строки более наглядны, что позволяет
использовать четкие фильтры с опциями __get__ из коробки в __REST__-
фреймворках).
* переменные для хранения вариантов являются константами. Поэтому
они должны быть указаны в верхнем регистре.
* Указывайте варианты перед списками полей.
* если это список статусов, указывайте их в хронологическом порядке
(например, new, __in_progress__, __completed__).
* вы можете использовать __Choices__ из библиотеки __model_utils__. Возьмем, к
примеру, модель __Article__:

```python
from model_utils import Choices

class Article(models.Model):
    STATUSES = Choices(
        (0, 'draft', _('draft')),
        (1, 'published', _('published')) )

    status = models.IntegerField(choices=STATUSES, default=STATUSES.draft)
    ...
```

#### 7. Много флагов в модели?
Если это обосновано, замените несколько __BooleanFields__ на одно поле,
подобное статусу. Например:

```python
class Article(models.Model):
    is_published = models.BooleanField(default=False)
    is_verified = models.BooleanField(default=False)
    ...
```
Предположим, что логика нашего приложения предполагает, что статья изначально не публикуется и не проверяется, затем она проверяется и помечается __is_verified__ в __True__, а затем публикуется. Вы можете заметить, что статья не может быть опубликована без проверки. Таким образом, всего есть 3 условия, но с двумя логическими полями у нас нет 4 возможных вариантов, и вы должны убедиться, что нет статей с неправильными
комбинациями условий логических полей. Вот почему использование одного поля статуса вместо двух булевых полей является лучшим вариантом:

```python
class Article(models.Model):
    STATUSES = Choices('new', 'verified', 'published')
    status = models.IntegerField(choices=STATUSES, default=STATUSES.draft)
    ...
```

Этот пример может быть не очень наглядным, но представьте, что в вашей
модели есть 3 или более таких булевых полей, и контроль валидации для
этих комбинаций значений полей может быть действительно
утомительным.


#### 8. Не складывайте все загруженные пользователем файлы в одну папку. 
Иногда даже отдельной папки для каждого поля FileField будет недостаточно, если ожидается большое количество загруженных файлов. Хранение большого количества файлов в одной папке означает, что файловая система будет искать нужный файл медленнее. Чтобы избежать подобных проблем, вы можете сделать следующее:

```python
def get_upload_path(instance, filename):
    return os.path.join('account/avatars/', now().date().strftime("%Y/%m/%d"), filename)

class User(AbstractUser):
    avatar = models.ImageField(blank=True, upload_to=get_upload_path)
```


#### 9. Используйте абстрактные модели 
Если вы хотите разделить некоторую логику между моделями, вы можете
использовать абстрактные модели.

```python
class CreatedatModel(models.Model):
    created_at = models.DateTimeField(
    verbose_name=u "Created at",
    auto_now_add=True
)

class Meta:
    abstract = True

class Post(CreatedatModel):
    ...

class Comment(CreatedatModel):
    ...
```


#### 10. Использование пользовательского менеджера и QuerySet 
Чем больше проект, над которым вы работаете, тем чаще вы повторяете один и тот же код в разных местах.

Чтобы сохранить код DRY и распределить бизнес-логику в моделях, вы можете использовать пользовательские менеджеры и Queryset.

```python
class CustomManager(models.Manager):
    def with_comments_counter(self):
        return self.get_queryset().annotate(comments_count=Count('comment_set'))
```

Теперь вы можете использовать:

```python
posts = Post.objects.with_comments_counter()
posts[0].comments_count
```

Если вы хотите использовать этот метод в связке с другими методами
__queryset__, вам следует использовать пользовательский __QuerySet__:

```python
class CustomQuerySet(models.query.QuerySet):
""""
Замена QuerySet, и добавление дополнительных методов к QuerySet.
"""
def with_comments_counter(self):
    """
    Добавляет счетчик комментариев к набору запросов
    """
    return self.annotate(comments_count=Count('comment_set'))
```

Теперь вы можете использовать:

```python
posts = Post.objects.filter(...).with_comments_counter()
posts[0].comments_count
```
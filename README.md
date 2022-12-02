#### How to work prefetch_related
---
#### models:

```
class Category(models.Model):
    name = models.CharField(max_length=255, unique=True)

    def __str__(self):
        return self.name

    class Meta:
        verbose_name = 'Category'
        verbose_name_plural = 'Categories'


class Tag(models.Model):
    name = models.CharField(max_length=255, unique=True)

    def __str__(self):
        return self.name


class Blog(models.Model):
    title    = models.CharField(max_length=255)
    slug     = models.SlugField(blank=True, null=True, unique=True)
    category = models.ForeignKey(Category, on_delete=models.SET_NULL, blank=True, null=True)
    body     = models.TextField(blank=True, null=True)
    tags     = models.ManyToManyField(Tag, blank=True)
    author   = models.ForeignKey(User, default=1, on_delete=models.SET_DEFAULT)
    created  = models.DateTimeField(auto_now=False, auto_now_add=True)
    updated  = models.DateTimeField(auto_now=True, auto_now_add=False)

    def __str__(self):
        return self.title


class Communities(models.Model):
    name       = models.CharField(max_length=255)
    blog_posts = models.ManyToManyField(Blog, blank=True)

    def __str__(self):
        return self.name

    class Meta:
        verbose_name = 'Community'
        verbose_name_plural = 'Communities'
```
---
#### without prefetch_related:

```
@query_debugger
def cld():
    qs = Communities.objects.all()

    print(qs.query)

    communities = []
    for item in qs:
        blog_posts = [post.title for post in item.blog_posts.all()]

        communities.append({
            'id': item.id,
            'name': item.name,
            'blog_posts': blog_posts,
        })

    return communities

```

```
>>> cld()
SELECT "blog_communities"."id", "blog_communities"."name" FROM "blog_communities"
Queries quantity: 21
Execution time: 0.37s

```

---
#### prefetch_related v1:

```
@query_debugger
def cld():
    qs = Communities.objects.prefetch_related('blog_posts')

    print(qs.query)

    communities = []
    for item in qs:
 
        blog_posts = [post.title for post in item.blog_posts.all()] 

        communities.append({
            'id': item.id,
            'name': item.name,
            'blog_posts': blog_posts,
        })

    return communities
```
```
>>> cld()
SELECT "blog_communities"."id", "blog_communities"."name" FROM "blog_communities"
Queries quantity: 2
Execution time: 0.35s
```
---
#### prefetch_related v2 (bad):

```
@query_debugger
def cld():

    qs = Communities.objects.prefetch_related('blog_posts')

    print(qs.query)

    communities = []
    for item in qs:
        blog_posts = [post.title for post in item.blog_posts.filter(tags__name__in=['Тег 1', 'Тег 2', 'Тег 3', 'Тег 4', 'Тег 5', 'Тег 6'])]

        communities.append({
            'id': item.id,
            'name': item.name,
            'blog_posts': blog_posts, # Prefetch
        })

    return communities
```
```
>>> cld()
SELECT "blog_communities"."id", "blog_communities"."name" FROM "blog_communities"
Queries quantity: 22
Execution time: 0.96s

```
---
#### prefetch_related v3:

```
@query_debugger
def cld():

    qs = Communities.objects.prefetch_related(
        Prefetch('blog_posts', queryset=Blog.objects.filter(tags__name__in=['Тег 1', 'Тег 2', 'Тег 3', 'Тег 4', 'Тег 5', 'Тег 6']))
    )

    print(qs.query)

    communities = []
    for item in qs:

        blog_posts = [post.title for post in item.blog_posts.all()]

        communities.append({
            'id': item.id,
            'name': item.name,
            'blog_posts': blog_posts, # Prefetch
        })

    return communities

```
```
>>> cld()
SELECT "blog_communities"."id", "blog_communities"."name" FROM "blog_communities"
Queries quantity: 2
Execution time: 0.68s

```

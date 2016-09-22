
http://www.machinalis.com/blog/nested-resources-with-django/ 

Nested resources with Django REST Framework
A complete example of how to implement nested resources with Django REST Framework

Posted by Agustín Bartó 11 months, 2 weeks ago Comments
Introduction
Django REST Framework (DRF) is a great way to provide RESTful interfaces for Django sites. Although it is fairly complete and does pretty much everything for you, it doesn’t cover all the possible use cases. One of those cases is nested resources. Whether nested resources are good or bad is a matter for a long discussion, but sadly sometimes an API design is forced on us and we have no choice but to try to match the specification as close as possible.

DRF’s documentation suggests using the @list-route and @detail-route decorators to solve this problem, but with this solution you lose most of the automatic functionality that makes DRF such a great tool.

This blogpost shows you how to implement an alternative solution for nested resources in Django REST framework using drf-nested-routers.

The code
The code for this blogpost is available on GitHub. A Vagrant configuration file is included if you want to test the service yourself.

drf-nested-routers
drf-nested-routers is a package that allows you to nest DRF’s routers within each other, effectively providing nested resources. Although it works like a charm, the example provided in the documentation assumes that developer already knows all the ins and outs of DRF and that’s not always the case (it wasn’t for me at least). I wrote this blogpost to provide a more thorough example.

The API
The model for the API describes blogposts and comments on those blogposts:

class UUIDIdMixin(models.Model):
    class Meta:
        abstract = True

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)


class AuthorMixin(models.Model):
    class Meta:
        abstract = True

    author = models.ForeignKey(
        settings.AUTH_USER_MODEL, editable=False, verbose_name=_('author'),
        related_name='%(app_label)s_%(class)s_author'
    )


class Blogpost(UUIDIdMixin, TimeStampedModel, TitleSlugDescriptionModel, AuthorMixin):
    content = models.TextField(_('content'), blank=True, null=True)
    allow_comments = models.BooleanField(_('allow comments'), default=True)


class Comment(UUIDIdMixin, TimeStampedModel, AuthorMixin):
    blogpost = models.ForeignKey(
        Blogpost, editable=False, verbose_name=_('blogpost'), related_name='comments'
    )
    content = models.TextField(_('content'), max_length=255, blank=False, null=False)
We want to expose the API for comments related to a specific blogpost on a path like /blogposts/<blogpost_id>/comments/. This can be solved in two ways:

We expose all the actions (list, retrieve, create, update, partial update and destroy) in nested paths like /blogposts/<blogpost_id>/comments/ (for list and create) and /blogposts/<blogpost_id>/comments/<comment_id> (for retrieve, update, partial update and destroy).
We provide list and create functionality in a path nested with the blogpost like /blogposts/<blogpost_id>/comments/, and the rest of the actions in a root path of its own like /comments for list and /comments/<comment_id> for retrieve, update, partial update and destroy.
Although I’m not a fan of nested resources (I prefer flat APIs with filters), the second approach is more in line with the principles of RESTful APIs and, as I’ll show you is easier to implement.

We want to expose two sets of actions under different paths and DRF provides an abstraction for such use case: a ViewSet. One of the great things about this framework is that it provides a lot of functionality in easy to use mixins. If we combine these two aspects, our solution is actually quite simple:

class BlogpostViewSet(ModelViewSet):
    serializer_class = BlogpostSerializer
    queryset = Blogpost.objects.all()
    permission_classes = (IsAuthenticatedOrReadOnly, IsAuthorOrReadOnly)

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


class CommentViewSet(
    RetrieveModelMixin, UpdateModelMixin, DestroyModelMixin, ListModelMixin, GenericViewSet
):
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = (IsAuthenticatedOrReadOnly, CommentDeleteOrUpdatePermission)


class NestedCommentViewSet(CreateModelMixin, ListModelMixin, GenericViewSet):
    queryset = Comment.objects.all()
    serializer_class = CommentSerializer
    permission_classes = (IsAuthenticatedOrReadOnly, CommentsAllowed)

    def get_blogpost(self, request, blogpost_pk=None):
        """
 Look for the referenced blogpost
 """
        # Check if the referenced blogpost exists
        blogpost = get_object_or_404(Blogpost.objects.all(), pk=blogpost_pk)

        # Check permissions
        self.check_object_permissions(self.request, blogpost)

        return blogpost

    def create(self, request, *args, **kwargs):
        self.get_blogpost(request, blogpost_pk=kwargs['blogpost_pk'])

        return super().create(request, *args, **kwargs)

    def perform_create(self, serializer):
        serializer.save(
            author=self.request.user,
            blogpost_id=self.kwargs['blogpost_pk']
        )

    def get_queryset(self):
        return Comment.objects.filter(blogpost=self.kwargs['blogpost_pk'])

    def list(self, request, *args, **kwargs):
        self.get_blogpost(request, blogpost_pk=kwargs['blogpost_pk'])

        return super().list(request, *args, **kwargs)
The first ViewSet is just a regular ModelViewSet to provide a typical API for the Blogpost model. The other two implement the design that I mentioned before, one ViewSet for the nested comments (NestedCommentViewSet) that only provides list and create actions (through the CreateModelMixin and ListModelMixin mixins) and a root level ViewSet for the rest of the actions.

You’ll also need the appropriate serializers:

class BlogpostSerializer(HyperlinkedModelSerializer):
    class Meta:
        model = Blogpost
        fields = ('url', 'title', 'slug', 'description', 'content', 'allow_comments', 'author', 'created', 'modified')
        read_only_fields = ('url', 'slug', 'author', 'created', 'modified')


class CommentSerializer(HyperlinkedModelSerializer):
    class Meta:
        model = Comment
        fields = ('url', 'content', 'author', 'created', 'modified', 'blogpost')
        read_only_fields = ('url', 'author', 'created', 'modified', 'blogpost')
All that is left is wiring these viewsets to URLs, and that’s when drf-nested-routers comes into play:

router = DefaultRouter()
router.register(r'users', UserViewSet)
router.register(r'blogposts', BlogpostViewSet)
router.register(r'comments', CommentViewSet)

blogposts_router = NestedSimpleRouter(router, r'blogposts', lookup='blogpost')
blogposts_router.register(r'comments', NestedCommentViewSet)

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^api/', include(router.urls)),
    url(r'^api/', include(blogposts_router.urls)),
    url(r'^o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
]
And that’s it. DRF will take care of the rest (no pun intended). The other approach would have required a bit of extra work like implementing a custom HyperlinkedRelatedField implementation.

Usage
We used OAuth2 for authentication and authorization, and created an application to allow access to the API. The application was defined as “Public” with grant type “Resource owner password-base”, so all we need to do to access the API is request an access token:

$ curl --silent --header "Content-Type: application/x-www-form-urlencoded" --data "username=admin&password=admin&grant_type=password&client_id=7ytbv0sG9FusDdDYRcZPUIGoNrx9TBZJnye5CVvj" --request POST http://localhost:8000/o/token/|python -mjson.tool; echo
{
    "access_token": "Q8Wbo12h5jwgwR208WDNrhNpK20Ta0",
    "expires_in": 36000,
    "refresh_token": "inEHtHuerVRXSH5QjbvokgrqJYxngL",
    "scope": "read write",
    "token_type": "Bearer"
}
Afterwards, you can request a list of blogposts:

$ curl --header "Authorization: Bearer Q8Wbo12h5jwgwR208WDNrhNpK20Ta0" --header "Accept: application/json; indent=4"  --request GET http://localhost:8000/api/blogposts/; echo
[
    {
        "url": "http://127.0.0.1:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/",
        "title": "A longer blogpost",
        "slug": "a-longer-blogpost",
        "description": "Lorem ipsum dolor sit amet...",
        "content": "Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus maximus, lorem eget accumsan maximus, ante mauris lacinia massa, sit amet pellentesque nisl leo eu libero. Fusce hendrerit risus eu vehicula cursus. Duis tincidunt enim eget felis tempus, ut consequat purus elementum.",
        "allow_comments": true,
        "author": "http://127.0.0.1:8000/api/users/2/",
        "created": "2015-07-10T00:15:38.135000Z",
        "modified": "2015-07-10T00:16:34.192000Z"
    },
    {
        "url": "http://127.0.0.1:8000/api/blogposts/b44d4918-219e-4496-9318-b68ab13e2b25/",
        "title": "A short blogpost",
        "slug": "a-short-blogpost",
        "description": "The description of the blogpost is short",
        "content": "This is just a short blogpost.",
        "allow_comments": true,
        "author": "http://127.0.0.1:8000/api/users/2/",
        "created": "2015-07-10T00:14:06.500000Z",
        "modified": "2015-07-10T00:14:06.501000Z"
    }
]
You can request a list of comments for a specific blogpost:

$ curl --header "Authorization: Bearer Q8Wbo12h5jwgwR208WDNrhNpK20Ta0" --header "Accept: application/json; indent=4"  --request GET http://localhost:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/comments/; echo
[
    {
        "url": "http://127.0.0.1:8000/api/comments/17288f69-bbd7-4758-adfd-a96d0fa5ca01/",
        "content": "I hate the Internet",
        "author": "http://127.0.0.1:8000/api/users/2/",
        "created": "2015-07-10T00:24:47.766000Z",
        "modified": "2015-07-10T00:24:47.766000Z",
        "blogpost": "http://127.0.0.1:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/"
    }
]
You can create create a new comment POSTing to the nested URL for a specific blogpost:

$ curl --verbose --header "Authorization: Bearer Q8Wbo12h5jwgwR208WDNrhNpK20Ta0" --header "Accept: application/json; indent=4" --header "Content-Type: application/json" --request POST --data '{"content": "No RESTful for the wicked"}' http://localhost:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/comments/; echo
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8000 (#0)
> POST /api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/comments/ HTTP/1.1
> User-Agent: curl/7.40.0
> Host: localhost:8000
> Authorization: Bearer Q8Wbo12h5jwgwR208WDNrhNpK20Ta0
> Accept: application/json; indent=4
> Content-Type: application/json
> Content-Length: 40
>
* upload completely sent off: 40 out of 40 bytes
< HTTP/1.1 201 CREATED
< Server: nginx/1.4.6 (Ubuntu)
< Date: Wed, 29 Jul 2015 18:15:30 GMT
< Content-Type: application/json; indent=4
< Transfer-Encoding: chunked
< Connection: keep-alive
< Vary: Accept
< Allow: GET, POST, HEAD, OPTIONS
< Location: http://127.0.0.1:8000/api/comments/81e5afc6-56fc-47d4-9665-db56229d0fba/
< X-Frame-Options: SAMEORIGIN
<
{
    "url": "http://127.0.0.1:8000/api/comments/81e5afc6-56fc-47d4-9665-db56229d0fba/",
    "content": "No RESTful for the wicked",
    "author": "http://127.0.0.1:8000/api/users/1/",
    "created": "2015-07-29T18:15:30.294242Z",
    "modified": "2015-07-29T18:15:30.294635Z",
    "blogpost": "http://127.0.0.1:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/"
* Connection #0 to host localhost left intact
}
You can verify that the comment was actually created by requesting the list of comments again:

$ curl --header "Authorization: Bearer Q8Wbo12h5jwgwR208WDNrhNpK20Ta0" --header "Accept: application/json; indent=4"  --request GET http://localhost:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/comments/; echo
[
    {
        "url": "http://127.0.0.1:8000/api/comments/17288f69-bbd7-4758-adfd-a96d0fa5ca01/",
        "content": "I hate the Internet",
        "author": "http://127.0.0.1:8000/api/users/2/",
        "created": "2015-07-10T00:24:47.766000Z",
        "modified": "2015-07-10T00:24:47.766000Z",
        "blogpost": "http://127.0.0.1:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/"
    },
    {
        "url": "http://127.0.0.1:8000/api/comments/81e5afc6-56fc-47d4-9665-db56229d0fba/",
        "content": "No RESTful for the wicked",
        "author": "http://127.0.0.1:8000/api/users/1/",
        "created": "2015-07-29T18:15:30.294242Z",
        "modified": "2015-07-29T18:15:30.294635Z",
        "blogpost": "http://127.0.0.1:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/"
    }
]
You can also hit the /comments endpoint directly to list, update or delete a comment:

$ curl --header "Authorization: Bearer Q8Wbo12h5jwgwR208WDNrhNpK20Ta0" --header "Accept: application/json; indent=4"  --request GET http://localhost:8000/api/comments/81e5afc6-56fc-47d4-9665-db56229d0fba/; echo
{
    "url": "http://127.0.0.1:8000/api/comments/81e5afc6-56fc-47d4-9665-db56229d0fba/",
    "content": "No RESTful for the wicked",
    "author": "http://127.0.0.1:8000/api/users/1/",
    "created": "2015-07-29T18:15:30.294242Z",
    "modified": "2015-07-29T18:15:30.294635Z",
    "blogpost": "http://127.0.0.1:8000/api/blogposts/588660f1-4848-4a32-8eb5-9688fd4409dd/"
}
Conclusions
Django REST Framework is powerful, flexible and easy to use. Although it doesn’t support nested resources, they can be implemented without much effort as long as you’re willing to accommodate certain design restrictions.


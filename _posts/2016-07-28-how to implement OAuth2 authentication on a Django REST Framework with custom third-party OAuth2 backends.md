http://www.machinalis.com/blog/oauth2-authentication/

OAuth2 authentication with Django REST Framework and custom third-party OAuth2 backends
A complete example of how to implement OAuth2 authentication on a Django REST Framework with custom third-party OAuth2 backends

Posted by Agustín Bartó 11 months, 2 weeks ago Comments
Introduction
Using OAuth2 authentication on a Django REST Framework based Django site it’s actually fairly easy to implement thanks to django-oauth-tookit. Things can be trick if you also want to allow the users of the API using third-party OAuth2 authentication backends like Facebook, Twitter, or even your own OAuth2 backends.

Authenticating with social sites on regular Django sites is actually quite easy thanks to applications like django-allauth, but sadly it is not well suited for RESTful APIs authentication.

Not to long ago, Félix Descôteaux wrote an excellent blogpost entitled “A Rest API using Django and authentication with OAuth2 AND third parties!” that showed us how to use third-party OAuth2 backends on Django REST Framework site using using django-oauth-toolkit.

This blogpost provides a complete example of an implementation of what Félix Descôteaux suggests, expanding on the matter by explaining how to use our own OAuth2 provider.

The code
The code for this blogpost is available on GitHub. A Vagrant configuration file is included if you want to test the service yourself.

Fake Social Site
Fake Social Site is a simple Django project with OAuth2 authentication (using django-oauth-toolkit) that will play the role of the third-party authentication provider. We could have used Facebook too, but we wanted to give you the chance to play with a couple of custom use cases that we had to tackle at Machinalis.

Besides OAuth2 authentication, the site provides two endpoints that expose the user’s profile.

user_details/: It returns the profile information for the currently authenticated user.
user_details_by_username/(?P<username>.+)/: It returns the profile information for a user with a given username. This enpoint was created to emulate the use case where the third-party site doesn’t offer an endpoint to show the info associated with the access_token. If the user making the request is different from the user referenced by the username or no users have the supplied username, we return a 404 response.
The following session illustrates the usage of these endpoints:

$ curl --header "Content-Type: application/x-www-form-urlencoded" --header "Accept: application/json; indent=4" --request POST --data "username=admin&password=admin&client_id=ZQcMr611iZMcUskTGoRcyZuhqCjZYy08lyOsWM5d&grant_type=password" http://localhost:8005/o/token/; echo
{"access_token": "zf8x8YiP3nUPjnV8WWArve4c3tZIMN", "token_type": "Bearer", "expires_in": 36000, "refresh_token": "fQF4BSp8nyFs72xobC2UzpeHHYmHYC", "scope": "read write"}

$ curl --head --header "Accept: application/json; indent=4" --request GET http://localhost:8005/user_details/; echo
HTTP/1.0 401 UNAUTHORIZED
Date: Fri, 03 Jul 2015 19:26:07 GMT
Server: WSGIServer/0.1 Python/2.7.10
Vary: Accept
X-Frame-Options: SAMEORIGIN
Content-Type: application/json; indent=4
WWW-Authenticate: Bearer realm="api"
Allow: OPTIONS, GET

$ curl --header "Authorization: Bearer zf8x8YiP3nUPjnV8WWArve4c3tZIMN" --header "Accept: application/json; indent=4" --request GET http://localhost:8005/user_details/; echo
{
    "id": 1,
    "username": "admin",
    "first_name": "Agustin",
    "last_name": "Barto",
    "email": "abarto@rest_oauth_social_test.com"
}

$ curl --header "Authorization: Bearer zf8x8YiP3nUPjnV8WWArve4c3tZIMN" --header "Accept: application/json; indent=4" --request GET http://localhost:8005/user_details_by_username/admin/; echo
{
    "id": 1,
    "username": "admin",
    "first_name": "Agustin",
    "last_name": "Barto",
    "email": "abarto@rest_oauth_social_test.com"
}

$ curl --head --header "Authorization: Bearer zf8x8YiP3nUPjnV8WWArve4c3tZIMN" --header "Accept: application/json; indent=4" --request GET http://localhost:8005/user_details_by_username/foobar/; echo
HTTP/1.0 404 NOT FOUND
Date: Fri, 03 Jul 2015 19:25:23 GMT
Server: WSGIServer/0.1 Python/2.7.10
Vary: Accept
X-Frame-Options: SAMEORIGIN
Content-Type: application/json; indent=4
Allow: OPTIONS, GET
We created two OAuth2 applications within the site: One for itself, and another that represents the other Django project that’ll serve as our example API (My API):

[
{
    "model": "oauth2_provider.application",
    "pk": 1,
    "fields": {
        "skip_authorization": true,
        "redirect_uris": "",
        "name": "my-api-app",
        "authorization_grant_type": "password",
        "client_type": "public",
        "client_id": "ZQcMr611iZMcUskTGoRcyZuhqCjZYy08lyOsWM5d",
        "client_secret": "Ve0AVjI4G7JwPj0spAz4jvY0nNxGGfK9q6IJXqARRS3oobDY0sYxqepH0i1euXDLfcbWe8Dx27atNMyJvg3vRLssUBJd4otkoNgxD6jwje5l3ipJnwGpNy3QFq0EhB1g",
        "user": [
            "admin"
        ]
    }
},
{
    "model": "oauth2_provider.application",
    "pk": 2,
    "fields": {
        "skip_authorization": true,
        "redirect_uris": "",
        "name": "fake-social-site-app",
        "authorization_grant_type": "password",
        "client_type": "public",
        "client_id": "aNARymvEsn21XPdpR9wJ8tPcUyto7rCu1ywo6H1T",
        "client_secret": "6Q7nFgQ8UExu2XhaNbI56DQAWTQvnVp7D9l3S9Ps1kpju3fH6NSRPfXktF92gZWmG1q7974NkuJSTm7nahTKdmaKcsaevA2U0tRjE4oXD66bqQrxDjQR8B7AK5JOM9Ko",
        "user": [
            "admin"
        ]
    }
}
]
My API
The second part of the project is a Django site that exposes a simple API using Django REST Framework and uses django-oauth-toolkit for authentication.

We want to allow users of Fake Social Site access to My API, as well as My API’s own user. As mentioned in the introduction we follow the recipe described in Félix Descôteaux’s blogpost (as well as python-social-auth’s documentation on the matter). The only change we made was to allow supplying custom parameters to the authentication backend when registering the user for the first time.

We expose a Django view that takes an OAuth2 access_token from Fake Social Site and exchanges it for one of My API, creating a new user and its social user profile in the process:

my_api/users/views.py:


_ERROR_TOKEN_EXCHANGE_RESPONSE = JsonResponse(
    {
        "error": "unsuccessful_token_exchange",
        "error_description": "Unable to complete token exchange with social backend."
    },
    status=401
)


@psa('social:complete')
def register_by_access_token(request, backend):
    token = request.GET.get('access_token')
    username = request.GET.get('username', None)

    # We pass the parameters to the backend so it can make the appropriate requests to the third party site.
    user = request.backend.do_auth(token, username=username)

    if user:
        try:
            login(request, user)
        except Exception:
            return _ERROR_TOKEN_EXCHANGE_RESPONSE
        else:
            return get_access_token(user)
    else:
        return _ERROR_TOKEN_EXCHANGE_RESPONSE

my_api/users/tools.py:

def get_token_json(access_token):
    return JsonResponse({
        'access_token': access_token.token,
        'expires_in': oauth2_settings.ACCESS_TOKEN_EXPIRE_SECONDS,
        'token_type': 'Bearer',
        'refresh_token': access_token.refresh_token.token,
        'scope': access_token.scope
    });


def get_access_token(user):
    application = Application.objects.get(name="my-api")

    try:
        old_access_token = AccessToken.objects.get(user=user, application=application)
        old_refresh_token = RefreshToken.objects.get(user=user, access_token=old_access_token)
    except:
        pass
    else:
        old_access_token.delete()
        old_refresh_token.delete()

    token = generate_token()
    refresh_token = generate_token()

    expires = now() + timedelta(seconds=oauth2_settings.ACCESS_TOKEN_EXPIRE_SECONDS)
    scope = "read write"

    access_token = AccessToken.objects.\
        create(user=user,
               application=application,
               expires=expires,
               token=token,
               scope=scope)

    RefreshToken.objects.\
        create(user=user,
               application=application,
               token=refresh_token,
               access_token=access_token)

    return get_token_json(access_token)
In the example of the blogpost, the author uses Facebook as the third party. In order to support Fake Social Site, we wrote an authentication backend based on python-social-backend’s BaseOAuth2:

class FakeSocialSiteOAuth2(BaseOAuth2):
    name = 'fake_social_site'
    SCOPE_SEPARATOR = ','
    EXTRA_DATA = [
        ('id', 'id')
    ]

    def access_token_url(self):
        return settings.FAKE_SOCIAL_SITE_AUTH_AUTHORIZATION_URL

    def authorization_url(self):
        return settings.FAKE_SOCIAL_SITE_AUTH_ACCESS_TOKEN_URL

    def get_user_details(self, response):
        return {
            'username': response.get('username'),
            'email': response.get('email') or '',
            'first_name': response.get('first_name'),
            'last_name': response.get('last_name'),
        }

    def user_data(self, access_token, *args, **kwargs):
        try:
            return self.get_json(
                settings.FAKE_SOCIAL_SITE_AUTH_USER_DETAILS_URL,
                headers={'Authorization': 'Bearer {}'.format(access_token)}
            )
        except ValueError:
            return None
There’s not much to it as we leveraged most of BaseOAuth2’s functionality. We also wanted to allow for the use case when the third party site requires a parameters to look for the user’s profile info, so we created another authentication provider based on BaseOAuth2:

class FakeSocialSiteWithParamsOAuth2(BaseOAuth2):
    name = 'fake_social_site_with_params'
    SCOPE_SEPARATOR = ','
    EXTRA_DATA = [
        ('id', 'id')
    ]

    def access_token_url(self):
        return settings.FAKE_SOCIAL_SITE_WITH_PARAM_AUTH_AUTHORIZATION_URL

    def authorization_url(self):
        return settings.FAKE_SOCIAL_SITE_WITH_PARAM_AUTH_ACCESS_TOKEN_URL

    def get_user_details(self, response):
        return {
            'username': response.get('username'),
            'email': response.get('email') or '',
            'first_name': response.get('first_name'),
            'last_name': response.get('last_name'),
        }

    def user_data(self, access_token, username=None, *args, **kwargs):
        try:
            return self.get_json(
                settings.FAKE_SOCIAL_SITE_WITH_PARAM_AUTH_USER_DETAILS_URL.format(username=username),
                headers={'Authorization': 'Bearer {}'.format(access_token)}
            )
        except ValueError:
            return None
All we had to do was add the named paratemeter (username in this case) to the user_data method, and use its value to make the request to the third party site. When the do_auth method is invoked in register_by_access_token we supply the parameter taken from the request, and it is passed to user_data when it is eventually invoked by python-social-auth’s authentication pipeline.

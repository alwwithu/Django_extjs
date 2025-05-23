# Combining ExtJS 5 with Django 1.6
Django is a wonderful framework on the Serverside and ExtJS ist a clear win for Client-Application-Developers. Would be nice if we could bring them together. This is a Proof of Concept of how this could be done. Far more then delivering code it is meant as Documentation and Reference. The Code is tested to run with python 3.3

To understand the explanations below, you should go through at least Part 1 & 2 of the [Django 1.6 Tutorial](https://docs.djangoproject.com/en/1.6/intro/tutorial01/). Instead of adding HTML-Templates in Part 3, we'll add a REST-Api and on top of that a nice ExtJS Application.

## Installing the dependencies
To install all dependencies Globally on your System, run

	sudo pip3 install -r requirements.txt

This will install

 - django 1.6 (stable at the time of writing)
 - djangorestframework (automatically expose django models as REST-Api-Calls)
 - django-filter (sorting and -optionally- filtering for the on the REST-Api)
 - django-rest-auth (login. logout and profile functions for the REST-Api)
 - django-registration (requirement of django-rest-auth)

## Setting up the REST-Api
The first thing we'll do is configuring the browsable REST-Api. For the existing Model [Polls](polls/models.py#L5) this will automatically generate views and routes to handle HTTP-Requests like this:

 - GET /api/polls (get a list of all polls)
 - POST /api/polls (add a new poll)
 - OPTIONS /api/polls (get meta-information of the Polls-Model)
 - GET /api/polls/5 (get Details of Poll Number 5)
 - PUT/PATCH /api/polls/5 (modify Poll Number 5)
 - DELETE /api/polls/5 (remove Number 5)

and some more. This feature is provided by the [Django REST framework](http://www.django-rest-framework.org/).
To enable this functionality, add the two apps `rest_framework` and `rest_framework.authtoken` to the [INSTALLED_APPS config](djangotest/settings.py#L32).

`rest_framework.authtoken` provides an Token-Based authorization (your Desktop-Client-Program saves a token instead of the original username/password) which we don't need here but the `rest_auth`-package does not work without.

Further down in the settings.py-File the [REST_FRAMEWORK configuration](djangotest/settings.py#L58) follows:

 - `DEFAULT_MODEL_SERIALIZER_CLASS` names the class that converts model-information to json. [rest_framework.serializers.HyperlinkedModelSerializer](http://www.django-rest-framework.org/tutorial/5-relationships-and-hyperlinked-apis) is the default class, that returns json based on the model definition but with hyperlinks to the clickable REST-Api
 - `DEFAULT_AUTHENTICATION_CLASSES` names the class that controlls how the current user is determined. [rest_framework.authentication.SessionAuthentication](http://www.django-rest-framework.org/api-guide/authentication#sessionauthentication) uses a session-cookie to identify the current user
 - DEFAULT_PERMISSION_CLASSES names  the class that controlls which user can access which information. [rest_framework.permissions.DjangoModelPermissions](http://www.django-rest-framework.org/api-guide/permissions#djangomodelpermissions) uses the permissions that can be controlled in the django admin-gui. Depending on your requirements there are [other defaults](http://www.django-rest-framework.org/api-guide/permissions) you should check out.
   ![django permission GUI](http://i.stack.imgur.com/icrtr.png)
 - `PAGINATE_BY_PARAM` configures the name of the parameter, that the django REST-Api expects from ExtJS to specify the number of results that should be returned. `limit` is what ExtJS assumes by default.
 - `ORDERING_PARAM` does the same for sorting. ExtJS defaults to `sort`
 - `PAGINATE_BY` is the default number of items to display per page. ExtJS has its own setting, so this setting does not really matter much
 - `MAX_PAGINATE_BY` limits the number of items that can be returned per page.
 - `DEFAULT_FILTER_BACKENDS` enables sorting and filtering by default. The classes named in this array are ties between the datasource an the REST-Api-Output and that can modify the queries sent to the database. The order of items in this array matters - first filter, then sort. This would be the place to add other filter backends, see "Filtering" below.

And finally a global setting - `LOGIN_REDIRECT_URL` - sets the url where a non-ajax user gets redirected after the login. This only matters when a user loggs in to the browsable api, because normal users will always use the Ajax-Gui. Setting this value to `/api/` is just a cosmetic thing.

## ViewSet, Serializer, Routers
In Django, each URL is connected to a View which determines the response sent when a request for that URL is received. In [Part 3 of the Tutorial](https://docs.djangoproject.com/en/1.6/intro/tutorial03/) you would create some views and connect them with Urls.

In the example above we see a set of different URLs that play together to form the REST-Api: /api/polls and /api/polls/5 each with a set of different HTTP Verbs. The Django REST-Api offers so-called ViewSets, which represent a Set of those Views. A ViewSet is specified around a set of rows queried from a Django-Model in [rest.py](polls/rest.py#L12) - a file I created to hold all the rest-related code. By extending your ViewSet-Class you can controll, which rows from your Database will be accessable on the REST-Api (ie only the completed tasks or only the published projects - whatever you need). You can do this by specifying filters when setting the queryset-Parameter.

The ViewSet is connected to a serializer class which specifies, which fields should be made visible on the REST-Api and how the rows will be represented. In our test-setup we just list the fields that should be visible on the api, but much more can be done in the Serializer, for example [embedding 1-to-n related rows](http://www.django-rest-framework.org/api-guide/relations).

Finally the Views presented by the ViewSet needs to be converted to URLs. This is done by the Router. The REST-Api works best with a single REST-Router, so we start in the [global urls.py-File](djangotest/urls.py#L9) and call into the register-method of the Polls-app in [Line 13](djangotest/urls.py#L13). The register-Method is defined in the [rest.py File](polls/rest.py#L16) of the Polls-App and just registers the ViewSet with the Router. The Router generates the all required urls and is embedded in the global url-scheme below `/api` in [the global urls.py-File](djangotest/urls.py#L30). `/api` was chosen arbitrarily by me.

## REST-Api-Urls
After this you can access the browable REST-Api by accessing your local server at [http://localhost:8000/api/](http://localhost:8000/api/). In the top-right corner there is a login-button which allows you to log into the browable REST-Api and see restricted resources. The required URLs are registered in the global [urls.py-File](djangotest/urls.py#L24), too, as [http://localhost:8000/api/ui-auth/](http://localhost:8000/api/ui-auth/). `/api/ui-auth` was also chosen arbitrarily by me.

## REST-Auth-Api
The Django-Backend has an idea of user, password, login and session. It can manage those for us. The question is: how can an ExtJS-App ask the Django-Side if a given username/password combination is valid and retrieve an auth-token for it, how can it check if a given auth-token is still valid and make the Django-Backend forget the token when the user opts to log out.

We employ the [django-rest-auth-Package](https://github.com/Tivix/django-rest-auth/) for this task. It provides a [convenient set of URLs](https://github.com/Tivix/django-rest-auth/#api-endpoints-with-authentication) to do these kind of tasks. It can simply be added to the global url routing in [urls.py](djangotest/urls.py#L27) as `/api/auth/` (test ie. [http://localhost:8000/api/auth/user/](http://localhost:8000/api/auth/user/).

## Frontend
The whole HTML/CSS/JS-Code lives in a Django-App called [frontend/](frontend). This app has no models but it has [a view](frontend/views.py#L7) which is connected with a url in the [urls.py-File](frontend/urls.py#L5). The View is decorated with a `@ensure_csrf_cookie` decorator, which advises Django to always send a CSRF-Cookie. Wikipedia explains good, [what CSRF is](http://en.wikipedia.org/wiki/CSRF). The CSRF-Cookie is used by ExtJS later to authoize its Ajax-Calls - more on this later.

The frontend-App has a [sinlge html-template](frontend/templates/frontend/index.html) which generates the basic code sent to the Browser. For the purpose of this example we embed `ext-all-debug.js` and `ext-theme-neptune-all.css` from the CDN. In a production-app we would build us a stripped-down version of extjs, combine it with our application and server everything from our own server.

Our ExtJS Application is located in [the static-folder](frontend/static/frontend) of our frontend-App. The main file is [frontend.js](frontend/static/frontend/frontend.js) which declares our Application and links to all the bits and pieces of the ExtJS Application.

## ExtJS MVC
The hardest thing when developing Django with ExtJS is to separate the two applications we're writing in your brain. ExtJS comes with its own Models, Views and Controllers, each in its own file and strictly named.

We start by setting the App-Name and the path to load source-files from (only iduring development, later all is baked into a single file). Then we tell ExtJS to load the CSRF-Token-Helper (see below) and our Application-Controllers.
 - The Login-Controller is responsible to check if and which user is logged into Django and to perform login or logout-tasks on the backend.
 - The Navigation-Controller provides the Base GUI consisting of a tree on the left side and a main container on the right. Application-Views add themselfs to both panels. Selecting a View-Name in the tree switches the right container to the selected Application-View.
 - The Polls-Controller is such an Application-View that displays an editable Grid with Polls



## Login
The [Login-Controller](frontend/static/frontend/controller/Login.js) handles login and logout procedure with the Django-backend, displays and handles the login-window. It is called very early from the [main js-file](frontend/static/frontend/frontend.js), which calls [ensureLoggedIn](frontend/static/frontend/controller/Login.js#L89) with a callback. The callback will be called when the Login-Controller made sure that a user is logged in.

The Login-Controller first calls `/api/auth/user/`, one of the apis provided by the [django-rest-auth-Package](https://github.com/Tivix/django-rest-auth/), to determine if the user accessing the application has a valid authorization token known by Django. If she has, the callback is called immedately and the applications boots up.

If no valid token was detected, the [login-view](frontend/static/frontend/view/Login.js) is shown. When it gets submitted by the user, the Login-Controller passes the entered information to the `/api/auth/login/` API, which accepts or denys the login credentials. When the user is granted access, the Login-Controller repeats its call to `/api/auth/user/` to get the updated user information and calls back as if the first call to `/api/auth/user/` had succeeded.

## Navigation
The [Navigation-Controller](frontend/static/frontend/view/Navigation.js) provides the main application view (in words: tree left, content right). The content-container to the right is configured to use [a card-layout](http://docs-origin.sencha.com/extjs/5.0.0/apidocs/#!/api/Ext.layout.container.Card), which displays one content view at a time (and hides all the other).

The Controller keeps track of all available content views by adding them to the navigation-tree and the content-view. The Content-Controllers adds their views to the Navigation-Controller which adds an item to the Navigation-Tree and the View to the content-container. When an item is selected in the navigation tree, the corresponding view in the content panel is shown.

## Demo-View: Polls
As an example I created the [Polls-Controller](frontend/static/frontend/controller/Polls.js) and [-View](frontend/static/frontend/view/Polls.js). The controller only adds the View to the Navigation-Controller as explained above. The View only consists of a single [GridPanel](http://docs-origin.sencha.com/extjs/5.0.0/apidocs/#!/api/Ext.grid.Panel) which is configured with paging, sorting and editing. It is bound to the [Polls-Store](frontend/static/frontend/store/Polls.js) which configures, how data is loaded from and saved to the Django-Backend.

## CrsfTokenHelper
The Django REST-Api reqires, that all None-GET-Requests carry an [CSRF-Token](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29_Prevention_Cheat_Sheet#Double_Submit_Cookies). The Token is sent to the Browser as a Cookie and must be copied to the POST-Parameters or the Request-Headers. Requests that do not fulfil this reqirement are rejected.

The [CrsfTokenHelper-Class](frontend/static/frontend/helper/CrsfTokenHelper.js) is loaded from `frontend.js`. It is declared as singleton and thus immedeatly instanciated by ExtJS. In its constructor it adds an `beforerequest`-Event-Handler to the ExtJS Ajax Routines, which statisfy the above mentioned need by copying the Cookie into a custem HTTP-Reqest-Header `X-CSRFToken`. Thus no other parts of the Application need to worry about CSRF Prevention Issues.

## DjangoProxy
Django has a very specific way of transporting the Sort-Direction over the API:

 - [http://localhost:8000/api/polls/?sort=question](http://localhost:8000/api/polls/?sort=question) sorts by question Ascending
 - [http://localhost:8000/api/polls/?sort=-question](http://localhost:8000/api/polls/?sort=-question) sorts by question Descending

The Sort-Direction is specified by the minus-Sign before the fieldname. This is not directly supported by ExtJS, so we need custom code to implement serializing the ExtJS Sort-Information into the GET-Parameter Value.

Additionally - when updating Rows - ExtJS sends partitial updates by default (only the changed fields). The REST-Apis `PUT`-Method-Handler expects complete Records and fails if fields are missing. We correct this by telling ExtJS to send a `PATCH`-Request instead of `POST` for updates. Djangos The REST-Api handles `PATCH` like `POST` but without the check for completion.

Both code parts can be found in the [DjangoProxy](frontend/static/frontend/helper/DjangoProxy.js)-Class, which is directly derived from [Ext.data.proxy.Rest](http://docs.sencha.com/extjs/5.0.0/apidocs/#!/api/Ext.data.proxy.Rest), the REST-Adapter supplied by ExtJS.

## Filtering
is currently a little problematic while it should work as easy as sorting.

## Running with Python 3
While under Python 3.3 everything works out of the box, the rest-auth-Plugin has problems wit python 3.2. The reason is this is, that it still contains Python2 conform `u'string'` Unicode-Literals which were dropped in Python3 and [readded in Python 3.3](https://docs.python.org/3/whatsnew/3.3.html) for backwards compatibility. I replaced a bunch of `u'string'` occurencies with `'string'` to get it running on a Debian-Stable server, but instead we should just send them a [Pull Request](https://github.com/Tivix/django-rest-auth/pulls).

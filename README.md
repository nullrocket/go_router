# go_router
The goal of the go_router package is to simplify use of
[the `Router` in Flutter](https://api.flutter.dev/flutter/widgets/Router-class.html)
as specified by [the
`MaterialApp.router` constructor](https://api.flutter.dev/flutter/material/MaterialApp/MaterialApp.router.html).
By default, it requires an implementation of the
[`RouterDelegate`](https://api.flutter.dev/flutter/widgets/RouterDelegate-class.html) and
[`RouteInformationParser`](https://api.flutter.dev/flutter/widgets/RouteInformationParser-class.html)
classes. These two implementations themselves imply the
definition of a custom type to hold the app state that drives the creation of the
[`Navigator`](https://api.flutter.dev/flutter/widgets/Navigator-class.html).
You can read [an excellent blog post on these requirements on Medium](https://medium.com/flutter/learning-flutters-new-navigation-and-routing-system-7c9068155ade).
This separation of responsibilities allows the Flutter developer to implement a number of routing
and navigation policies at the cost of [complexity](https://www.reddit.com/r/FlutterDev/comments/koxx4w/why_navigator_20_sucks/).

The go_router makes three simplifying assumptions to reduce complexity:
1. all routing in the app will happen via schemeless absolute
   [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier)-compliant names
1. an entire stack of pages can be constructed from the route name alone
1. the concept of "back" in your app is "up" the stack of pages

These assumptions allow go_router to provide a simpler implementation of your app's custom router regardless
of the platform you're targeting. Specifically, since web users can enter arbitrary locations to navigate
your app, go_router is designed to be able to handle arbitrary locations while still allowing an easy-to-use
developer experience.

# Getting Started
To use the go_router package, [follow these instructions](https://pub.dev/packages/go_router/install).

# Declarative Routing
The go_router is governed by a set of routes which you specify via a routes builder function:

```dart
class App extends StatelessWidget {
  ...
  late final _router = GoRouter(routes: _routesBuilder, error: _errorBuilder);
  List<GoRoute> _routesBuilder(BuildContext context, String location) => [
        GoRoute(
          pattern: '/',
          builder: (context, args) => MaterialPage<FamiliesPage>(
            key: const ValueKey('FamiliesPage'),
            child: FamiliesPage(families: Families.data),
          ),
        ),
        GoRoute(
          pattern: '/family/:fid',
          builder: (context, args) {
            final family = Families.family(args['fid']!);

            return MaterialPage<FamilyPage>(
              key: ValueKey(family),
              child: FamilyPage(family: family),
            );
          },
        ),
        GoRoute(
          pattern: '/family/:fid/person/:pid',
          builder: (context, args) {
            final family = Families.family(args['fid']!);
            final person = family.person(args['pid']!);

            return MaterialPage<PersonPage>(
              key: ValueKey(person),
              child: PersonPage(family: family, person: person),
            );
          },
        ),
      ];
  ...
}
```

In this case, we've defined 3 routes. The route name patterns are defined and implemented in the
[`path_to_regexp`](https://pub.dev/packages/path_to_regexp)
package, which gives you the ability to include regular expressions, e.g. `/family/:fid(f\d+)`. These route name patterns will be matched in order and every pattern that matches a prefix of the
location will be a page on the navigation stack like so:

pattern                    | example location       | navigation stack
---------------------------|------------------------|-----------------
`/`                        | `/`                    | `FamiliesPage()`
`/family/:fid`             | `/family/f1`           | `FamiliesPage()`, `FamilyPage(f1)`
`/family/:fid/person/:pid` | `/family/f1/person/p2` | `FamiliesPage()`, `FamilyPage(f1)`, `PersonPage(p2)`

The order of the patterns in the list of routes dictates the order in the
navigation stack. The navigation stack is used to pop up to the previous page in
the stack when the user press the Back button or your app calls
`Navigation.pop()`.

In addition to the pattern, a `GoRoute` contains a page builder function which
is called to create the page when a pattern is matched. That function can use
the arguments parsed from the pattern to do things like look up data to use
to initialize each page.

In addition, the go_router needs an error handler in case no page is found or
if any of the page builder functions throws an exception, e.g.

```dart
class App extends StatelessWidget {
  ...
  late final _router = GoRouter(routes: _routesBuilder, error: _errorBuilder);
  ...
  Page<dynamic> _errorBuilder(BuildContext context, GoRouteException ex) => MaterialPage<Four04Page>(
        key: const ValueKey('Four04Page'),
        child: Four04Page(message: ex.nested.toString()),
      );
}
```

The `GoRouteException` object contains the location that caused the exception and a nested `Exception`
that was thrown attempting to navigate to that route.

With these two functions in hand, you can establish your app's custom routing
policy using the `MaterialApp.router` constructor:

```dart
class App extends StatelessWidget {
  App({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) => MaterialApp.router(
        routeInformationParser: _router.routeInformationParser,
        routerDelegate: _router.routerDelegate,
      );

  late final _router = GoRouter(routes: _routesBuilder, error: _errorBuilder);
  List<GoRoute> _routesBuilder(BuildContext context, String location) => ...
  Page<dynamic> _errorBuilder(BuildContext context, GoRouteException ex) => ...
}
```

With the router in place, your app can now navigate between pages.

# Navigation
To navigate between pages, use the `GoRouter.go` method:

```dart
// navigate using the GoRouter
onTap: () => GoRouter.of(context).go('/family/f1/person/p2')
```

The go_router also provides a simplified version using Dart extension methods:

```dart
// more easily navigate using the GoRouter
onTap: () => context.go('/family/f1/person/p2')
```

The simplified version maps directly to the more fully-specified version, so you can use either.

# URL Path Strategy
By default, Flutter adds a hash (#) into the URL for web apps:

![URL Strategy w/ Hash](readme/url-strat-hash.png)

The process for turning off the hash is [documented](https://flutter.dev/docs/development/ui/navigation/url-strategies)
but fiddly. The go_router has built-in support for setting the URL path strategy, however, so you can
simply call `GoRouter.setUrlPathStrategy` before calling `runApp` and make your choice:

```dart
void main() {
  // turn on the # in the URLs on the web (default)
  // GoRouter.setUrlPathStrategy(UrlPathStrategy.hash);

  // turn off the # in the URLs on the web
  GoRouter.setUrlPathStrategy(UrlPathStrategy.path);

  runApp(App());
}
```

Setting the path instead of the hash strategy turns off the # in the URLs:

![URL Strategy w/o Hash](readme/url-strat-no-hash.png)

Finally, when you deploy your Flutter web app to a web server, it needs to be configured such that every URL ends up
at your Flutter web app's `index.html`, otherwise Flutter won't be able to route to your pages.
If you're using Firebase hosting, you can [configure rewrites](https://firebase.google.com/docs/hosting/full-config#rewrites) to cause all URLs to be rewritten to `index.html`.

If you'd like to test locally before publishing, you can use [live-server](https://www.npmjs.com/package/live-server)
like so:

```sh
$ live-server --entry-file=index.html build/web
```

Of course, any local web server that can be configured to redirect all traffic to `index.html` will do.

# Conditional Routes
The routes builder is called each time that the location changes, which allows you to change the routes based on the
location. Furthermore, if you'd like to change the set of routes based on conditional app state, you can do so using
`InheritedWidget` or one of it's wrappers. For example, imagine a simple class to track the app's current logged in
state:

```dart
class LoginInfo extends ChangeNotifier {
  var _userName = '';
  bool get loggedIn => _userName.isNotEmpty;

  void login(String userName) {
    _userName = userName;
    notifyListeners();
  }
}
```

Because the `LoginInfo` is a `ChangeNotifier`, it can accept listeners and notify them of data changes.
We can then use [the provider package](https://pub.dev/packages/provider)
(which is based on `InheritedWidget`) to drop an instance of `LoginInfo` into the widget tree:

```dart
class App extends StatelessWidget {
  // add the login info into the tree as app state that can change over time
  @override
  Widget build(BuildContext context) => ChangeNotifierProvider<LoginInfo>(
        create: (context) => LoginInfo(),
        child: MaterialApp.router(
          routeInformationParser: _router.routeInformationParser,
          routerDelegate: _router.routerDelegate,
          title: 'Conditional Routes GoRouter Example',
        ),
      );
...
}
```

Now imagine a login page that pulls the login info out of the widget tree and changes the login state as appropriate:

```dart
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: Text(_title(context))),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton(
                // log a user in, letting all the listeners know
                onPressed: () => context.read<LoginInfo>().login('user1'),
                child: const Text('Login'),
              ),
            ],
          ),
        ),
      );
...
}
```

Notice the use of `context.read` from the provider package to walk the widget tree to find the login info and login a
sample user. This causes the listeners to this data to be notified and for any widgets listening for this change to
rebuild. We can then use this data when implementing the `GoRouter` routes builder to decide which routes are allowed:

```dart
class App extends StatelessWidget {
  ...
  // the routes when the user is logged in
  final _loggedInRoutes = [
    GoRoute(
      pattern: '/',
      builder: (context, args) => MaterialPage<FamiliesPage>(...),
    ),
    GoRoute(
      pattern: '/family/:fid',
      builder: (context, args) => MaterialPage<FamilyPage>(...),
    ),
    GoRoute(
      pattern: '/family/:fid/person/:pid',
      builder: (context, args) => MaterialPage<PersonPage>(...),
    ),
  ];

  // the routes when the user is not logged in
  final _loggedOutRoutes = [
    GoRoute(
      pattern: '/',
      builder: (context, args) => MaterialPage<LoginPage>(...),
    ),
  ];

  late final _router = GoRouter.routes(
    // changes in the login info will rebuild the stack of routes
    builder: (context, location) => context.watch<LoginInfo>().loggedIn ? _loggedInRoutes : _loggedOutRoutes,
    ...
  );
}
```

Here we've defined two lists of routes, one for when the user is logged in and one for when they're not. Then, we use
`context.watch` to read the login info to determine which list of routes to return. And because we used `context.watch`
instead of `context.read`, whenever the login info object changes, the routes builder is automatically called for the
correct list of routes based on the current app state.

# Redirection
Sometimes you want to redirect one route to another one, e.g. if the user is not logged in. You can do that using
an instance of the `GoRedirect` object, e.g.

```dart
List<GoRoute> _routesBuilder(BuildContext context, String location) => [
  GoRoute(
    pattern: '/',
    builder: (context, args) {
      final loggedIn = context.watch<LoginInfo>().loggedIn;
      if (!loggedIn) return const GoRedirect('/login');

      return MaterialPage<FamiliesPage>(
        key: const ValueKey('FamiliesPage'),
        child: FamiliesPage(families: Families.data),
      );
    },
  ),
  ...
  GoRoute(
    pattern: '/login',
    builder: (context, args) {
      final loggedIn = context.watch<LoginInfo>().loggedIn;
      if (loggedIn) return const GoRedirect('/');

      return const MaterialPage<LoginPage>(
        key: ValueKey('LoginPage'),
        child: LoginPage(),
      );
    },
  ),
];
```

In this code, if the user is not logged in, we redirect from the `/` to `/login`. Likewise, if the user *is* logged in,
we redirect from `/login` to `/`. And because we're using `context.watch`, when the login state changes, the routes
builder will be called again to generate and match the routes.

# Query Parameters
If you'd like to use query parameters for navigation, you can; they will be considered as optional for the purpose
of matching a route but passed along as arguments to the page builders. For example, if you'd like to redirect to
`/login` with the original location as a query parameter so that after a successful login, the user can be routed
back to the original location, you can do that using query paramaters:

```dart
List<GoRoute> _routesBuilder(BuildContext context, String location) => [
  ...
  GoRoute(
    pattern: '/family/:fid',
    builder: (context, args) {
      final loggedIn = context.watch<LoginInfo>().loggedIn;
      if (!loggedIn) return GoRedirect('/login?from=$location');

      final family = Families.family(args['fid']!);
      return MaterialPage<FamilyPage>(
        key: ValueKey(family),
        child: FamilyPage(family: family),
      );
    },
  ),
  ...
  GoRoute(
    pattern: '/login',
    builder: (context, args) {
      final loggedIn = context.watch<LoginInfo>().loggedIn;
      if (loggedIn) return const GoRedirect('/');

      return MaterialPage<LoginPage>(
        key: const ValueKey('LoginPage'),
        child: LoginPage(from: args['from']),
      );
    },
  ),
];
```

In this example, if the user isn't logged in, they're redirected to `/login` with a `from` query parameter set to the
original location. When the `/login` route is matched, the optional `from` parameter is passed to the `LoginPage`. In
the `LoginPage` if the `from` parameter was passed, we use it to go to the original location:

```dart
class LoginPage extends StatelessWidget {
  final String? from;
  const LoginPage({this.from, Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: Text(_title(context))),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton(
                // log a user in, letting all the listeners know
                onPressed: () {
                  context.read<LoginInfo>().login('user1');
                  if (from != null) context.go(from!);
                },
                child: const Text('Login'),
              ),
            ],
          ),
        ),
      );
}
```

A query parameter will not override a positional parameter or another query parameter set earlier in the location
string.

# Custom Builder
As described, the go_router uses the list of `GoRoute` objects to implement it's routing policy using patterns to
match and using the order of matches to create the `Navigator.pop()` implementation, etc. If you'd like to implement
the routing policy yourself, you can implement a lower level builder that is given a location and is responsible for
producing a `Navigator`. For example, this lower level builder can be implemented and passed to the
`GoRouter.builder` constructor like so:

```dart
class App extends StatelessWidget {
  ...
  late final _router = GoRouter.builder(builder: _builder);
  Widget _builder(BuildContext context, String location) {
    final locPages = <String, Page<dynamic>>{};

    try {
      final segments = Uri.parse(location).pathSegments;

      // home page, i.e. '/'
      {
        const loc = '/';
        final page = MaterialPage<FamiliesPage>(
          key: const ValueKey('FamiliesPage'),
          child: FamiliesPage(families: Families.data),
        );
        locPages[loc] = page;
      }

      // family page, e.g. '/family/:fid
      if (segments.length >= 2 && segments[0] == 'family') {
        final fid = segments[1];
        final family = Families.family(fid);

        final loc = '/family/$fid';
        final page = MaterialPage<FamilyPage>(
          key: ValueKey(family),
          child: FamilyPage(family: family),
        );

        locPages[loc] = page;
      }

      // person page, e.g. '/family/:fid/person/:pid
      if (segments.length >= 4 && segments[0] == 'family' && segments[2] == 'person') {
        final fid = segments[1];
        final pid = segments[3];
        final family = Families.family(fid);
        final person = family.person(pid);

        final loc = '/family/$fid/person/$pid';
        final page = MaterialPage<PersonPage>(
          key: ValueKey(person),
          child: PersonPage(family: family, person: person),
        );

        locPages[loc] = page;
      }

      // if we haven't found any matching routes OR
      // if the last route doesn't match exactly, then we haven't got a valid stack of pages;
      // the latter allows '/' to match as part of a stack of pages but to fail on '/nonsense'
      if (locPages.isEmpty || locPages.keys.last.toString().toLowerCase() != location.toLowerCase()) {
        throw Exception('page not found: $location');
      }
    } on Exception catch (ex) {
      locPages.clear();

      final loc = location;
      final page = MaterialPage<Four04Page>(
        key: const ValueKey('ErrorPage'),
        child: Four04Page(message: ex.toString()),
      );

      locPages[loc] = page;
    }

    return Navigator(
      pages: locPages.values.toList(),
      onPopPage: (route, dynamic result) {
        if (!route.didPop(result)) return false;

        // remove the route for the page we're showing and go to the next location up
        locPages.remove(locPages.keys.last);
        _router.go(locPages.keys.last);

        return true;
      },
    );
  }
}
```

There's a lot going on here, but it fundamentally boils down to 3 things:
1. Matching portions of the location to instances of the app's pages using manually parsed URI segments for
   arguments. This mapping is kept in an ordered map so it can be used as a stack of location=>page mappings.
1. Providing an implementation of `onPopPage` that will translate `Navigation.pop` to use the
   location=>page mappings to navigate to the previous page on the stack.
1. Show an error page if any of that fails.

This is the basic policy that the `GoRouter` itself implements, although in a simplified form w/o features like
route name patterns, redirection or query parameters.

# Examples
You can see the go_router in action via the following examples:
- [`routes.dart`](example/lib/routes.dart): define a routing policy but using a set of declarative `GoRoute` objects
- [`url_strategy.dart`](example/lib/url_strategy.dart): turn off the # in the Flutter web URL
- [`conditional.dart`](example/lib/conditional.dart): provide different routes based on changing app state
- [`redirection.dart`](example/lib/redirection.dart): redirect one route to another based on changing app state
- [`query_params.dart`](example/lib/query_params.dart): optional query parameters will be passed to all page builders
- [`builder.dart`](example/lib/builder.dart): define routing policy by providing a custom builder

You can run these examples from the command line like so:

```sh
$ flutter run example/lib/routes.dart
```

Or, if you're using Visual Studio Code, a [`launch.json`](.vscode/launch.json) file has been provided with
these examples configured.

# TODO
- add a section on Deep Linking: https://flutter.dev/docs/development/ui/navigation/deep-linking
- move the TODO items into the issues database
- test with dialogs and Navigator.pop() to make sure we didn't screw anything up there...
- add docs showing async id => object lookup
- add support for nested routing ala https://github.com/flutter/samples/pull/832
- custom transition support
- unit testing
- widget testing
- support for shorter locations that result in multiple pages for a single route, e.g. /person/:pid
  could end up mapping to three pages (home, families and person) but will only match two routes
  (home and person). The mapping to person requires two pages to be returned (families and person).
- ...
- profit!
- BUG: navigating back too fast crashes

---
layout: plugin-nav-bar
group: third-party
---

# Developing plugins with Firebase

The Firebase JavaScript client v1.0.17 and [AngularFire v{{site.angularFireVersion}}]({{site.baseurl}}/libraries/angularfire/{{site.angularFireVersion}}/){:target="_blank"} is available to all plugin developers. Use it to store and sync data in realtime.

This guide covers basic aspects on how to use Firebase in your plugins. For a complete reference or tutorials, checkout the links in the bottom of this page.

## Initializing Firebase

After creating an app in Firebase, go to the settings section in the Developer Tools and save the URL and secret of your Firebase app. To initialize Firebase from your plugin frontend, inject the `$firebase` service in the controllers, services, factories or directives signature, create a reference to your Firebase URL and assign it to the scope:

{% highlight js %}
/**
 * My Plugin Controller
 */
plugin.controller('myPluginCntl', ['$scope', '$firebase', function ($scope, $firebase) {
    
    var peopleRef = new Firebase('https://<my-firebase>.firebaseio.com/people');
    
    $scope.people = $firebase(peopleRef);
    
}])
{% endhighlight %}

## Adding data to a list

The `$add` method takes a single argument of any type. It will append this value as a member of a list.

{% highlight js %}
/**
 * My Plugin Controller
 */
plugin.controller('myPluginCntl', ['$scope', '$firebase', function ($scope, $firebase) {
    
    var peopleRef = new Firebase('https://<my-firebase>.firebaseio.com/people');
    
    $scope.people = $firebase(peopleRef).$asArray();
    
    $scope.people.$add({
        name: 'Steve',
        location: 'Philadelphia, US'
    });
    
}])
{% endhighlight %}

## Adding or Updating keys

The `$update` method takes a single argument of any type. It will append or update existing keys.

{% highlight js %}
/**
 * My Plugin Controller
 */
plugin.controller('myPluginCntl', ['$scope', '$firebase', function ($scope, $firebase) {

    // Initialize a reference direct to a person using the <id>
    var personRef = new Firebase('https://<my-firebase>.firebaseio.com/people/<id>');

    $scope.person = $firebase(personRef);

    // Add the person's birthday
    $scope.person.$update({
        birthday: '12/10/1980'
    });
    
    // Or change the person's location
    $scope.person.$update({
        location: 'Tokyo, JP'
    });
    
    
}])
{% endhighlight %}

## Removing data

The `$remove` method takes a single optional argument, a key. If a key is provided, this method will remove the child referenced by that key.

{% highlight js %}
/**
 * My Plugin Controller
 */
plugin.controller('myPluginCntl', ['$scope', '$firebase', function ($scope, $firebase) {
    
    // A reference direct to a person using the <id>
    var personRef = new Firebase('https://<my-firebase>.firebaseio.com/people/<id>');
    
    $scope.person = $firebase(personRef); 
    
    // Remove the person
    $scope.person.$remove();
    
    // Or remove only the location key
    $scope.person.$remove('location');
    
}])
{% endhighlight %}

## Authentication

To get advantage of the already logged in user in {{site.productName}} and authenticate this same user in Firebase, you need to use the [custom login](https://www.firebase.com/docs/security/custom-login.html) method, this method uses a JWT token instead of a username and password.

You can get a token to authenticate the current user in Firebase by fetching your plugin data using the `Data` factory. The response will contain a `firebaseAuthToken` attribute, which is generated using the Firebase secret that you set in your "Plugin Settings".

The example below demonstrates how you can fetch the current plugin and user data from {{site.productName}} API, and then use it to connect to Firebase.

{% highlight js %}
/**
 * My Plugin Controller
 */
plugin.controller('myPluginCntl', ['$scope', 'znData', '$firebase', function ($scope, znData,  $firebase) {

    /**
     * Get the current plugin data
     */
    znData('Plugins').get(
        {
            namespace: 'myPlugin'
        },
        function(resp) {
            $scope.plugin = resp[0];
        },
        function(resp) {
            $scope.err = resp;
        }
    );
    
    /**
     * Get the currently logged-in user data
     */
    znData('Users').get(
        {
            id: 'me',
        },
        function(resp) {
            $scope.user = resp;
        },
        function(resp) {
            $scope.err = resp;
        }
    );
    
    /**
     * Wait for plugin and user data and connects to Firebase
     */
    var unbind = $scope.$watchCollection('[plugin, user]', function() {
    
        // Stop on any errors fetching plugin or user data
        if ($scope.err) {
            return;
        }
        
        // Checks if both plugin and user data is available to connect
        if ($scope.plugin !== undefined && $scope.user !== undefined) {
            $scope.connect();
            unbind();
        }

    });
    
    /**
     * Connects to firebase and authenticate
     */
    $scope.connect = function() {
    
        // Uses the `$scope.user.id` to make a direct reference to the current user preferences
        var ref = new Firebase('https://<my-firebase>.firebaseio.com/preferences/' + $scope.user.id);
        
        // Uses the `$scope.plugin.firebaseAuthToken` to authenticate the user
        ref.auth($scope.plugin.firebaseAuthToken, function(err, res) {
            
            if (err) {
                $scope.err = err;
            }
            
            // Set auth data
            $scope.auth = res.auth;
            
            // Set preferences
            $scope.preferences = $firebase(ref).$asObject();
            
            // Apply changes to the scope
            $scope.$apply();
            
        });
    
    };
    
}])
{% endhighlight %}

## Security rules

Using the [Firebase dashboard](https://www.firebase.com/account/), you can setup security rules to define when your data can be read from and written to.

When using Firebase from the frontend Javascript of a plugin, the following data is passed to Firebase and made available with the `auth` variable in your security rules:

{% highlight json %}
{
    "user_id": "1",
    "workspaces": {
        1: "admin",
        2: "owner",
        3: "owner",
        4: "member"
    }
}
{% endhighlight %}

The `workspaces` object contains all of the workspaces the currently authenticated user is a member of, where each key is the workspace id and the corresponding value is the user's role in that workspace.

Now we'll show how you can use this `auth` variable in your security rules. In the example below, the reference `https://<my-firebase>.firebaseio.com/preferences/<user-id>` is a list of preferences of a specific user id. The rules below allow read and write access only if the currently authenticated user id matches the `<user-id>`.

{% highlight json %}
{
    "rules": {
        "preferences": {
            "$user": {
                ".read": "$user == auth.user_id",
                ".write": "$user == auth.user_id"
            }
        }
    }
}
{% endhighlight %}

Another example is restricting read access to any user belonging to the workspace and write access to workspace owners, assuming the following reference to `https://<my-firebase>.firebaseio.com/preferences/<workspace-id>`

{% highlight json %}
{
    "rules": {
        "preferences": {
            "$workspace": {
                ".read": "auth.workspaces[$workspace] != null",
                ".write": "auth.workspaces[$workspace] != null && auth.workspaces[$workspace] == 'owner'"
            }
        }
    }
}
{% endhighlight %}

When using the znFirebase wrapper with a [Backend service]({{site.baseurl}}/plugins/development/services.html), znFirebase will automatically authenticate and the following data will be available with the `auth` variable in your security rules, assuming the request was made to a backend service at `{{site.pluginDomain}}/workspaces/1/testBackendService/testRoute`:

{% highlight json %}
{
    "workspaces": {
        1: "server"
    }
}
{% endhighlight %}

Similar to the authentication data from the frontend, the `auth` data from the backend contains a `workspaces` object. However, for backend service Firebase authentication, the `workspaces` object contains a single key-value pair: the workspace id which the backend service request was made against and the value `server`.

With this, you can extend your security rules to also allow backend service access. We will expand on the previous example to also allow read and write access to backend services:

{% highlight json %}
{
    "rules": {
        "preferences": {
            "$workspace": {
                ".read": "auth.workspaces[$workspace] != null",
                ".write": "auth.workspaces[$workspace] == 'owner' || auth.workspaces[$workspace] == 'server'"
            }
        }
    }
}
{% endhighlight %}

Learn more about [Firebase security rules](https://www.firebase.com/docs/security/guide/index.html){:target="_blank"}.

## Links

* [Chat Tutorial]({{site.baseurl}}/plugins/tutorials/building-a-chat-plugin.html)
* [Get started](https://www.firebase.com/how-it-works.html){:target="_blank"}
* [AngularJS + Firebase](https://www.firebase.com/quickstart/angularjs.html){:target="_blank"}
* [AngularFire API reference v{{site.angularFireVersion}}]({{site.baseurl}}/libraries/angularfire/{{site.angularFireVersion}}/){:target="_blank"}
* [Javascript client API reference](https://www.firebase.com/docs/javascript/firebase/index.html){:target="_blank"}
* [Open Data Sets](https://www.firebase.com/docs/data/index.html){:target="_blank"}

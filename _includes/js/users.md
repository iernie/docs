# Users

At the core of many apps, there is a notion of user accounts that lets users access their information in a secure manner. We provide a specialized user class called `Parse.User` that automatically handles much of the functionality required for user account management.

With this class, you'll be able to add user account functionality in your app.

`Parse.User` is a subclass of `Parse.Object`, and has all the same features, such as flexible schema, automatic persistence, and a key value interface. All the methods that are on `Parse.Object` also exist in `Parse.User`. The difference is that `Parse.User` has some special additions specific to user accounts.

## `Parse.User` Properties

`Parse.User` has several values that set it apart from `Parse.Object`:

*   username: The username for the user (required).
*   password: The password for the user (required on signup).
*   email: The email address for the user (optional).

We'll go through each of these in detail as we run through the various use cases for users.

## Signing Up

The first thing your app will do is probably ask the user to sign up. The following code illustrates a typical sign up:

<pre><code class="javascript">
var user = new Parse.User();
user.set("username", "my name");
user.set("password", "my pass");
user.set("email", "email@example.com");

// other fields can be set just like with Parse.Object
user.set("phone", "415-392-0202");

user.signUp(null, {
  success: function(user) {
    // Hooray! Let them use the app now.
  },
  error: function(user, error) {
    // Show the error message somewhere and let the user try again.
    alert("Error: " + error.code + " " + error.message);
  }
});
</code></pre>

This call will asynchronously create a new user in your Parse App. Before it does this, it also checks to make sure that both the username and email are unique. Also, it securely hashes the password in the cloud using bcrypt. We never store passwords in plaintext, nor will we ever transmit passwords back to the client in plaintext.

Note that we used the `signUp` method, not the `save` method. New `Parse.User`s should always be created using the `signUp` method. Subsequent updates to a user can be done by calling `save`.

If a signup isn't successful, you should read the error object that is returned. The most likely case is that the username or email has already been taken by another user. You should clearly communicate this to your users, and ask them try a different username.

You are free to use an email address as the username. Simply ask your users to enter their email, but fill it in the username property &mdash; `Parse.User` will work as normal. We'll go over how this is handled in the reset password section.

## Logging In

Of course, after you allow users to sign up, you need to let them log in to their account in the future. To do this, you can use the class method `logIn`.

<pre><code class="javascript">
Parse.User.logIn("myname", "mypass", {
  success: function(user) {
    // Do stuff after successful login.
  },
  error: function(user, error) {
    // The login failed. Check error to see why.
  }
});
</code></pre>

## Verifying Emails

Enabling email verification in an application's settings allows the application to reserve part of its experience for users with confirmed email addresses. Email verification adds the `emailVerified` key to the `Parse.User` object. When a `Parse.User`'s `email` is set or modified, `emailVerified` is set to `false`. Parse then emails the user a link which will set `emailVerified` to `true`.

There are three `emailVerified` states to consider:

1.  `true` - the user confirmed his or her email address by clicking on the link Parse emailed them. `Parse.Users` can never have a `true` value when the user account is first created.
2.  `false` - at the time the `Parse.User` object was last refreshed, the user had not confirmed his or her email address. If `emailVerified` is `false`, consider calling `fetch` on the `Parse.User`.
3.  _missing_ - the `Parse.User` was created when email verification was off or the `Parse.User` does not have an `email`.

## Current User

It would be bothersome if the user had to log in every time they open your app. You can avoid this by using the cached current `Parse.User` object.

Whenever you use any signup or login methods, the user is cached in localStorage. You can treat this cache as a session, and automatically assume the user is logged in:

<pre><code class="javascript">
var currentUser = Parse.User.current();
if (currentUser) {
    // do stuff with the user
} else {
    // show the signup or login page
}
</code></pre>

You can clear the current user by logging them out:

<pre><code class="javascript">
Parse.User.logOut().then(() => {
  var currentUser = Parse.User.current();  // this will now be null
});
</code></pre>

## Setting the Current User

If you’ve created your own authentication routines, or otherwise logged in a user on the server side, you can now pass the session token to the client and use the `become` method. This method will ensure the session token is valid before setting the current user.

<pre><code class="javascript">
Parse.User.become("session-token-here").then(function (user) {
  // The current user is now set to user.
}, function (error) {
  // The token could not be validated.
});
</code></pre>

## Security For User Objects

The `Parse.User` class is secured by default. Data stored in a `Parse.User` can only be modified by that user. By default, the data can still be read by any client. Thus, some `Parse.User` objects are authenticated and can be modified, whereas others are read-only.

Specifically, you are not able to invoke any of the `save` or `delete` methods unless the `Parse.User` was obtained using an authenticated method, like `logIn` or `signUp`. This ensures that only the user can alter their own data.

The following illustrates this security policy:

<pre><code class="javascript">
var user = Parse.User.logIn("my_username", "my_password", {
  success: function(user) {
    user.set("username", "my_new_username");  // attempt to change username
    user.save(null, {
      success: function(user) {
        // This succeeds, since the user was authenticated on the device

        // Get the user from a non-authenticated method
        var query = new Parse.Query(Parse.User);
        query.get(user.objectId, {
          success: function(userAgain) {
            userAgain.set("username", "another_username");
            userAgain.save(null, {
              error: function(userAgain, error) {
                // This will error, since the Parse.User is not authenticated
              }
            });
          }
        });
      }
    });
  }
});

</code></pre>

The `Parse.User` obtained from `Parse.User.current()` will always be authenticated.

If you need to check if a `Parse.User` is authenticated, you can invoke the `authenticated` method. You do not need to check `authenticated` with `Parse.User` objects that are obtained via an authenticated method.

## Security For Other Objects

The same security model that applies to the `Parse.User` can be applied to other objects. For any object, you can specify which users are allowed to read the object, and which users are allowed to modify an object. To support this type of security, each object has an [access control list](http://en.wikipedia.org/wiki/Access_control_list), implemented by the `Parse.ACL` class.

The simplest way to use a `Parse.ACL` is to specify that an object may only be read or written by a single user. This is done by initializing a Parse.ACL with a `Parse.User`: `new Parse.ACL(user)` generates a `Parse.ACL` that limits access to that user. An object's ACL is updated when the object is saved, like any other property. Thus, to create a private note that can only be accessed by the current user:

<pre><code class="javascript">
var Note = Parse.Object.extend("Note");
var privateNote = new Note();
privateNote.set("content", "This note is private!");
privateNote.setACL(new Parse.ACL(Parse.User.current()));
privateNote.save();
</code></pre>

This note will then only be accessible to the current user, although it will be accessible to any device where that user is signed in. This functionality is useful for applications where you want to enable access to user data across multiple devices, like a personal todo list.

Permissions can also be granted on a per-user basis. You can add permissions individually to a `Parse.ACL` using `setReadAccess` and `setWriteAccess`. For example, let's say you have a message that will be sent to a group of several users, where each of them have the rights to read and delete that message:

<pre><code class="javascript">
var Message = Parse.Object.extend("Message");
var groupMessage = new Message();
var groupACL = new Parse.ACL();

// userList is an array with the users we are sending this message to.
for (var i = 0; i < userList.length; i++) {
  groupACL.setReadAccess(userList[i], true);
  groupACL.setWriteAccess(userList[i], true);
}

groupMessage.setACL(groupACL);
groupMessage.save();
</code></pre>

You can also grant permissions to all users at once using `setPublicReadAccess` and `setPublicWriteAccess`. This allows patterns like posting comments on a message board. For example, to create a post that can only be edited by its author, but can be read by anyone:

<pre><code class="javascript">
var publicPost = new Post();
var postACL = new Parse.ACL(Parse.User.current());
postACL.setPublicReadAccess(true);
publicPost.setACL(postACL);
publicPost.save();
</code></pre>

Operations that are forbidden, such as deleting an object that you do not have write access to, result in a `Parse.Error.OBJECT_NOT_FOUND` error code. For security purposes, this prevents clients from distinguishing which object ids exist but are secured, versus which object ids do not exist at all.

## Resetting Passwords

It's a fact that as soon as you introduce passwords into a system, users will forget them. In such cases, our library provides a way to let them securely reset their password.

To kick off the password reset flow, ask the user for their email address, and call:

<pre><code class="javascript">
Parse.User.requestPasswordReset("email@example.com", {
  success: function() {
  // Password reset request was sent successfully
  },
  error: function(error) {
    // Show the error message somewhere
    alert("Error: " + error.code + " " + error.message);
  }
});
</code></pre>

This will attempt to match the given email with the user's email or username field, and will send them a password reset email. By doing this, you can opt to have users use their email as their username, or you can collect it separately and store it in the email field.

The flow for password reset is as follows:

1.  User requests that their password be reset by typing in their email.
2.  Parse sends an email to their address, with a special password reset link.
3.  User clicks on the reset link, and is directed to a special Parse page that will allow them type in a new password.
4.  User types in a new password. Their password has now been reset to a value they specify.

Note that the messaging in this flow will reference your app by the name that you specified when you created this app on Parse.

## Querying

To query for users, you can simple create a new `Parse.Query` for `Parse.User`s:

<pre><code class="javascript">
var query = new Parse.Query(Parse.User);
query.equalTo("gender", "female");  // find all the women
query.find({
  success: function(women) {
    // Do stuff
  }
});
</code></pre>

## Associations

Associations involving a `Parse.User` work right of the box. For example, let's say you're making a blogging app. To store a new post for a user and retrieve all their posts:

<pre><code class="javascript">
var user = Parse.User.current();

// Make a new post
var Post = Parse.Object.extend("Post");
var post = new Post();
post.set("title", "My New Post");
post.set("body", "This is some great content.");
post.set("user", user);
post.save(null, {
  success: function(post) {
    // Find all posts by the current user
    var query = new Parse.Query(Post);
    query.equalTo("user", user);
    query.find({
      success: function(usersPosts) {
        // userPosts contains all of the posts by the current user.
      }
    });
  }
});
</code></pre>

## Facebook Users

Parse provides an easy way to integrate Facebook with your application. The `Parse.FacebookUtils` class integrates `Parse.User` and the Facebook Javascript SDK to make linking your users to their Facebook identities easy.

Using our Facebook integration, you can associate an authenticated Facebook user with a `Parse.User`. With just a few lines of code, you'll be able to provide a "log in with Facebook" option in your app, and be able to save their data to Parse.


### Setting up Facebook

To start using Facebook with Parse, you need to:

1.  [Set up a Facebook app](https://developers.facebook.com/apps), if you haven't already.  Choose the "Website with Facebook Login" option under "Select how your app integrates with Facebook" and enter your site's URL.
2.  Add your application's Facebook Application ID on your Parse application's settings page.
3.  Follow [these instructions](https://developers.facebook.com/docs/javascript/quickstart/) for loading the Facebook JavaScript SDK into your application.
4.  Replace your call to `FB.init()` with a call to `Parse.FacebookUtils.init()`. For example, if you load the Facebook JavaScript SDK asynchronously, your `fbAsyncInit` function will look like this:

<pre><code class="javascript">
&lt;script&gt;
  // Initialize Parse
  Parse.initialize("$PARSE_APPLICATION_ID", "$PARSE_JAVASCRIPT_KEY");

      window.fbAsyncInit = function() {
    Parse.FacebookUtils.init({ // this line replaces FB.init({
      appId      : '{facebook-app-id}', // Facebook App ID
      status     : true,  // check Facebook Login status
      cookie     : true,  // enable cookies to allow Parse to access the session
      xfbml      : true,  // initialize Facebook social plugins on the page
      version    : 'v2.3' // point to the latest Facebook Graph API version
    });

        // Run code after the Facebook SDK is loaded.
  };

      (function(d, s, id){
    var js, fjs = d.getElementsByTagName(s)[0];
    if (d.getElementById(id)) {return;}
    js = d.createElement(s); js.id = id;
    js.src = "//connect.facebook.net/en_US/sdk.js";
    fjs.parentNode.insertBefore(js, fjs);
  }(document, 'script', 'facebook-jssdk'));
&lt;/script&gt;
</code></pre>

The function assigned to `fbAsyncInit` is run as soon as the Facebook JavaScript SDK has completed loading. Any code that you want to run after the Facebook JavaScript SDK is loaded should be placed within this function and after the call to `Parse.FacebookUtils.init()`.

If you encounter any issues that are Facebook-related, a good resource is the [official getting started guide from Facebook](https://developers.facebook.com/docs/reference/javascript/).

If you encounter issues that look like they're being returned from Parse's servers, try removing your Facebook application's App Secret from your app's settings page.

There are two main ways to use Facebook with your Parse users: (1) logging in as a Facebook user and creating a `Parse.User`, or (2) linking Facebook to an existing `Parse.User`.

### Login & Signup

`Parse.FacebookUtils` provides a way to allow your `Parse.User`s to log in or sign up through Facebook. This is accomplished using the `logIn()` method:

<pre><code class="javascript">
Parse.FacebookUtils.logIn(null, {
  success: function(user) {
    if (!user.existed()) {
      alert("User signed up and logged in through Facebook!");
    } else {
      alert("User logged in through Facebook!");
    }
  },
  error: function(user, error) {
    alert("User cancelled the Facebook login or did not fully authorize.");
  }
});
</code></pre>

When this code is run, the following happens:

1.  The user is shown the Facebook login dialog.
2.  The user authenticates via Facebook, and your app receives a callback.
3.  Our SDK receives the Facebook data and saves it to a `Parse.User`. If it's a new user based on the Facebook ID, then that user is created.
4.  Your `success` callback is called with the user.

You may optionally provide a comma-delimited string that specifies what [permissions](https://developers.facebook.com/docs/authentication/permissions/) your app requires from the Facebook user.  For example:

<pre><code class="javascript">
Parse.FacebookUtils.logIn("user_likes,email", {
  success: function(user) {
    // Handle successful login
  },
  error: function(user, error) {
    // Handle errors and cancellation
  }
});
</code></pre>

`Parse.User` integration doesn't require any permissions to work out of the box (ie. `null` or specifying no permissions is perfectly acceptable). [Read more about permissions on Facebook's developer guide.](https://developers.facebook.com/docs/reference/api/permissions/)

<div class='tip info'><div>
  It is up to you to record any data that you need from the Facebook user after they authenticate. To accomplish this, you'll need to do a graph query using the Facebook SDK.
</div></div>


### Linking

If you want to associate an existing `Parse.User` to a Facebook account, you can link it like so:

<pre><code class="javascript">
if (!Parse.FacebookUtils.isLinked(user)) {
  Parse.FacebookUtils.link(user, null, {
    success: function(user) {
      alert("Woohoo, user logged in with Facebook!");
    },
    error: function(user, error) {
      alert("User cancelled the Facebook login or did not fully authorize.");
    }
  });
}
</code></pre>

The steps that happen when linking are very similar to log in. The difference is that on successful login, the existing `Parse.User` is updated with the Facebook information. Future logins via Facebook will now log the user into their existing account.

If you want to unlink Facebook from a user, simply do this:

<pre><code class="javascript">
Parse.FacebookUtils.unlink(user, {
  success: function(user) {
    alert("The user is no longer associated with their Facebook account.");
  }
});
</code></pre>

### Facebook SDK and Parse

The Facebook Javascript SDK provides a main `FB` object that is the starting point for many of the interactions with Facebook's API. [You can read more about their SDK here](https://developers.facebook.com/docs/reference/javascript/).

Facebook login using the Parse SDK requires that the Facebook SDK already be loaded before calling `Parse.FacebookUtils.init()`.

Our library manages the `FB` object for you. The `FB` singleton is synchronized with the current user by default, so any methods you call on it will be acting on the Facebook user associated with the current `Parse.User`. Calling `FB.login()` or `FB.logOut()` explicitly will cause the `Parse.User` and `FB` object to fall out of synchronization, and is not recommended.

## Linking Users

Parse allows you to link your users with services like Twitter and Facebook, enabling your users to sign up or log into your application using their existing identities. This is accomplished through \_linkWith() method by providing authentication data for the service you wish to link to a user in the `authData` field. Once your user is associated with a service, the `authData` for the service will be stored with the user and is retrievable by logging in.

`authData` is a JSON object with keys for each linked service containing the data below. In each case, you are responsible for completing the authentication flow (e.g. OAuth 1.0a using hellojs on client side) to obtain the information the the service requires for linking.

### Facebook `authData`

<pre><code class="javascript">
{
  "facebook": {
    "id": "user's Facebook id number as a string",
    "access_token": "an authorized Facebook access token for the user",
    "expiration_date": "token expiration date of the format: yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"
  }
}
</code></pre>
Learn more about [Facebook login](https://developers.facebook.com/docs/authentication/).

### Twitter `authData`

<pre><code class="javascript">
{
  "twitter": {
    "id": "user's Twitter id number as a string",
    "screen_name": "user's Twitter screen name",
    "consumer_key": "your application's consumer key",
    "consumer_secret": "your application's consumer secret",
    "auth_token": "an authorized Twitter token for the user with your application",
    "auth_token_secret": "the secret associated with the auth_token"
  }
}
</code></pre>

Learn more about [Twitter login](https://dev.twitter.com/docs/auth/implementing-sign-twitter).

### Anonymous user `authData`

<pre><code class="javascript">
{
  "anonymous": {
    "id": "random UUID with lowercase hexadecimal digits"
  }
}
</code></pre>

### Signing Up and Logging In

Signing a user up with a linked service and logging them in with that service uses the same \_linkWith() mothod, in which the `authData` for the user is specified.  For example, to sign up or log in with a user's Twitter account:

<pre><code class="javascript">
let myAuthData = {
  "id": "12345678",
  "screen_name": "ParseIt",
  "consumer_key": "SaMpLeId3X7eLjjLgWEw",
  "consumer_secret": "SaMpLew55QbMR0vTdtOACfPXa5UdO2THX1JrxZ9s3c",
  "auth_token": "12345678-SaMpLeTuo3m2avZxh5cjJmIrAfx4ZYyamdofM7IjU",
  "auth_token_secret": "SaMpLeEb13SpRzQ4DAIzutEkCE2LBIm2ZQDsP3WUU"
}
let user = new Parse.User();
user._linkWith('twitter', myAuthData).then(function(user){
    // user
});
</code></pre>
Parse then verifies that the provided `authData` is valid and checks to see if a user is already associated with this data.  If so, it returns a status code of `200 OK` and the details (including a `sessionToken` for the user):

<pre><code class="javascript">
Status: 200 OK
Location: https://api.parse.com/1/users/uMz0YZeAqc
</code></pre>

With a response body like:

<pre><code class="json">
{
  "username": "Parse",
  "createdAt": "2012-02-28T23:49:36.353Z",
  "updatedAt": "2012-02-28T23:49:36.353Z",
  "objectId": "uMz0YZeAqc",
  "sessionToken": "r:samplei3l83eerhnln0ecxgy5",
  "authData": {
    "twitter": {
      "id": "12345678",
      "screen_name": "ParseIt",
      "consumer_key": "SaMpLeId3X7eLjjLgWEw",
      "consumer_secret": "SaMpLew55QbMR0vTdtOACfPXa5UdO2THX1JrxZ9s3c",
      "auth_token": "12345678-SaMpLeTuo3m2avZxh5cjJmIrAfx4ZYyamdofM7IjU",
      "auth_token_secret": "SaMpLeEb13SpRzQ4DAIzutEkCE2LBIm2ZQDsP3WUU"
    }
  }
}
</code></pre>
If the user has never been linked with this account, you will instead receive a status code of `201 Created`, indicating that a new user was created:

<pre><code class="javascript">
Status: 201 Created
Location: https://api.parse.com/1/users/uMz0YZeAqc
</code></pre>
The body of the response will contain the `objectId`, `createdAt`, `sessionToken`, and an automatically-generated unique `username`.  For example:

<pre><code class="json">
{
  "username": "iwz8sna7sug28v4eyu7t89fij",
  "createdAt": "2012-02-28T23:49:36.353Z",
  "objectId": "uMz0YZeAqc",
  "sessionToken": "r:samplei3l83eerhnln0ecxgy5"
}
</code></pre>

### Linking

Linking an existing user with a service like Facebook or Twitter uses the same method \_linkWith() to associate `authData` with the user.  For example, linking a user with a Facebook account would use a request like this:

<pre><code class="javascript">
let myAuthData = {
  id: "123456789",
  "access_token": "SaMpLeAAibS7Q55FSzcERWIEmzn6rosftAr7pmDME10008bWgyZAmv7mziwfacNOhWkgxDaBf8a2a2FCc9Hbk9wAsqLYZBLR995wxBvSGNoTrEaL",
  "expiration_date": "2012-02-28T23:49:36.353Z"
}
let user = new Parse.User();
user.id = "uMz0YZeAqc";
user._linkWith("facebook", myAuthData).then(function(user){
  // user is linked now
});
</code></pre>
After linking your user to a service, you can authenticate them using matching `authData`.

### Unlinking

Unlinking an existing user with a service also uses \_linkWith() method to clear `authData` from the user by setting the `authData` for the service to `null`.  For example, unlinking a user with a Facebook account would use a request like this:

<pre><code class="javascript">
let user = new Parse.User();
user.id = "uMz0YZeAqc";
user._linkWith("facebook", {}).then(function(user){
  // user is unlinked now
});
</code></pre>

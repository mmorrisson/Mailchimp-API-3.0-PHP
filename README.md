# MAILCHIMP API 3.0 PHP [![Build Status](https://travis-ci.com/Jhut89/Mailchimp-API-3.0-PHP.svg?branch=master)](https://travis-ci.com/Jhut89/Mailchimp-API-3.0-PHP)

This is a PHP library for interacting with [version 3.0 of MailChimp's API](https://developer.mailchimp.com)

This library assumes a basic understanding of the MailChimp application and its associated functions. 

## Installation

For Composer run:

```php
composer require jhut89/mailchimp3php
```

Alternatively you may add a require line to your projects `composer.json` for the package `jhut89/mailchimp3php`.

Then run `composer update` and add composer autoloader to your project with:

```php
require "vendor/autoload.php";
```
## Instantiation

```php
$mailchimp = new Mailchimp('123abc123abc123abc123abc-us0');
```

To instantiate you will need a new instance of the `Mailchimp` class with your MailChimp account's API key as its only argument.

## Oauth

If you are using [Oauth](http://developer.mailchimp.com/documentation/mailchimp/guides/how-to-use-oauth2/) to obtain an access token, this library can handle the "handshake" for you.
 
You must first send the user to your applications `authorize_uri`. You can get this url by calling the `Mailchimp::getAuthUrl()` statically:

```php 
$client_id =   '12345676543';
$redirect_url =  'https://www.some-domain.com/callback_file.php';

Mailchimp::getAuthUrl($client_id, $redirect_url);
```

From there the user will input their username and password to approve your application and will be redirected to the `redirect_uri` you set along with a `code`.

With that `code` you can now request an access token from mailchimp. You will need to call the `MAilchimp::oauthExchange()` method statically like this:

```php
$code = 'abc123abc123abc123abc123';
$client_id =   '12345676543';
$client_secret =  '789xyz789xyz789xyz789xyz';
$redirect_url =  'https://www.some-domain.com/callback_file.php';

Mailchimp::oauthExchange($code, $client_id, $client_secret, $redirect_url);
```

If the handshake is successful, then this method will return a string containing your API key like this: `123abc123abc123abc123abc123abc-us0`. This API key can now be used to instantiate your `Mailchimp` class as we have above.


## Constructing a Request

Once you have instantiated the `Mailchimp` class you can start constructing requests. Constructing requests is done by 'chaining' methods to the `$mailchimp` instance. In most cases this 'chain' will end with the HTTP verb for your request. So an example of retrieving a lists collection would look like this:

```php
$mailchimp
    ->lists()
    ->get();
```

Retrieving an instance can be accomplished by giving a unique identifier for the instance you want as an argument to the appropriate method. For example if I wanted to retrieve a list instance from the above example I would simply pass a `list_id`, as the only argument for the `lists()` method. Like this:

```php
$mailchimp
    ->lists('1a2b3c4d')
    ->get();
```

Methods available for each position in the chain depend on what the prior method returns. For example if I wanted to retrieve subscribers from a list in my account I would:

```php
$mailchimp
    ->lists('1a2b3c4d')
    ->members()
    ->get();
```

Notice that I provided a `list_id` to the `lists()` method, as there would be no way to retrieve a list of subscribers from a lists collection. The above request however will only return 10 subscriber instances from the members collection. This is because MailChimp's API uses pagination (documented [HERE](http://developer.mailchimp.com/documentation/mailchimp/guides/get-started-with-mailchimp-api-3/#parameters)) that defaults to `count=10` and `offset=0`. This library allows you to alter query string parameters by by passing them as an argument to the `GET()` method. We do this by providing an array of key-value pairs where the keys are the query parameter you wish to provide/alter and its value is the parameter's value. As an example if I wanted to retrieve the second 100 subscribers from my list I could:

```php
$mailchimp
    ->lists('1a2b3c4d')
    ->members()
    ->get([
        "count" => "100", 
        "offset" => "100"
    ]);
```

This would be equivalent to making a get request against:

```
Https://us0.api.mailchimp.com/3.0/lists/1a2b3c4d/members?count=100&offset=100
```

Going a little further we can retrieve a single list member by giving the `members_hash` (md5 hash of lower-case address) to the `members()` method. Like this:

```php
$mailchimp
    ->lists('1a2b3c4d')
    ->members('8bdbf060209f35b52087992a3cbdf4d7')
    ->get();
```

### Handling A Response

Methods named for http verbs such as `get()` ,`post()`, `patch()`, `put()`, or `delete()` kick off an over the wire request to MailChimp's A.P.I. Given a successful request these methods return an instance of a `MailchimpResponse`. I suggest you become familiar with this class as there a a number of modifiers and getters for different pieces of a response.

There are a number of getters we can use to interact with pieces our `MailchimpResponse` instance. Some of the more commonly used ones are:

```php
$response->deserialize(); // returns a deserialized (to php object) resource returned by API
$response->getRaw(); // return the raw text response
$response->getHttpCode(); // returns an integer representation of the HTTP response code
$response->getHeaders(); // returns an array of headers in key => value format
$response->getBody(); // return the raw text body of the response
```

As an example, if I have an API key but want the contact email associated with its account I would:

```php
$mailchimp = new Mailchimp('123abc123abc123abc123abc-us0');
$account = $mailchimp
    ->account()
    ->get()
    
$email = $account
    ->deserialize()
    ->email
    
print $email; // outputs something like "example@domain.com"
```

### POST

While being able to retrieve data from your account is great we also need to be able to post new data. This can be done by calling the `POST()` method at the end of a chain. As an example subscribing an address to a list would look like this:

```php
$post_params = [
    'email_address'=>'example@domain.com', 
    'status'=>'subscribed'
];

$mailchimp
    ->lists('1a2b3c4d')
    ->members()
    ->post($post_params);
```

In this case I would not provide `members()` with an identifier as I want to post to its collection. Also notice that the post data is an array of key-value pairs representing what parameters I want to pass to the MailChimp API. Be sure that you provide all required fields for the endpoint you are posting to. Check [MailChimp's documentation](http://developer.mailchimp.com/documentation/mailchimp/reference/lists/members/#create-post_lists_list_id_members) for what parameters are required. Non-required parameters can just be added to the post data, and MailChimp will ignore any that are unusable. To illustrate here is an example of adding a subscriber to a list with some non-required parameters:

```php
$merge_values = [
    "FNAME" => "John",
     "LNAME" => "Doe"
];

$post_params = [
    "email_address" => "example@domain.com", 
    "status" => "subscribed", 
    "email_type" => "html", 
    "merge_fields" => $merge_values
]

$mailchimp
    ->lists('1a2b3c4d')
    ->members()
    ->post($post_params);
```

### PATCH/PUT

This library handles PUT and PATCH request similar to that of POST requests. Meaning that `PUT()` & `PATCH()` both accept an array of key-value pairs that represent the data you wish altered/provided to MailChimp. As an example if I was patching the subscriber that we subscribed above, to have a new first name, that would look like this.

```php
$mailchimp
    ->lists('1a2b3c4d')
    ->members('a1167f5be2df7113beb69c95ebcdb2fd')
    ->patch([ "merge_fields" => ["FNAME" => "Jane"]]);
```

### DELETE

Deleting a record from MailChimp is performed with the `DELETE()` method and is constructed similar to GET requests. If I wanted to delete the above subscriber I would:

```php
$mailchimp
    ->lists('1a2b3c4d')
    ->members('a1167f5be2df7113beb69c95ebcdb2fd')
    ->delete();
```

## Method Chart (\*excluding verbs)

                                    
      mailchimp()                       
      |                                 
      |----account()                    
      |                                 
      |----apps()                       
      |                                 
      |----automations()                
      |    |                            
      |    |----removedSubscribers()    
      |    |----emails()                
      |         |                       
      |         |---queue()             
      |         |---pauseAll()         
      |         |---startAll()         
      |                                 
      |----batches()                    
      |                                 
      |----batchWebhooks()              
      |                                 
      |----campaignFolders()            
      |                                 
      |----campaigns()                  
      |    |                            
      |    |----cancel()                
      |    |----pause()                 
      |    |----replicate()             
      |    |----resume()                
      |    |----scedule()              
      |    |----send()                  
      |    |----test()                  
      |    |----unschedule()            
      |    |----checklist()             
      |    |----feedback()              
      |    |----content()               
      |                                 
      |----conversations()              
      |    |                            
      |    |----messages()              
      |                                 
      |----ecommStores()                
      |    |                            
      |    |----customers()             
      |    |----products()              
      |    |    |                       
      |    |    |----variants()         
      |    |    |----images()           
      |    |                            
      |    |----orders()                
      |    |    |                       
      |    |    |----lines()            
      |    |                            
      |    |----carts()                 
      |         |                       
      |         |----lines()            
      |                                 
      |----fileManagerFiles()           
      |                                 
      |----fileManagerFolders()         
      |                                 
      |----lists()                      
      |    |                            
      |    |----batchSubscribe()             
      |    |----webhooks()              
      |    |----signupForms()           
      |    |----mergeFields()           
      |    |----growthHistory()         
      |    |----clients()               
      |    |----activity()              
      |    |----abuseReports()          
      |    |----segments()              
      |    |    |                       
      |    |    |----batch()            
      |    |    |----members()          
      |    |                            
      |    |----members()               
      |    |    |                       
      |    |    |---notes()             
      |    |    |---goals()             
      |    |    |---activity()          
      |    |                            
      |    |----interestCategories()    
      |         |                       
      |         |----interests()        
      |                                 
      |----reports()                    
      |    |                            
      |    |----unsubscribes()          
      |    |----subReports()            
      |    |----sentTo()                
      |    |----locations()             
      |    |----emailActivity()         
      |    |----eepurl()                
      |    |----domainPerformance()     
      |    |----advice()                
      |    |----abuse()                 
      |    |----clickReports()          
      |         |                       
      |         |----members()          
      |                                 
      |----searchCampaigns()            
      |                                 
      |----searchMembers()              
      |                                 
      |----templateFolders()            
      |                                 
      |----templates()                  
           |                            
           |----defaultContent()        
                                    
                                    
                                    
                                    

\*Please see [MailChimp's API Documentation](http://developer.mailchimp.com/documentation/mailchimp/reference/overview/) for what verbs are appropriate where.

## Settings

This library offers several setting that can be altered by changing the value of the class constants at the begining of the `Mailchimp` class in the `mailchimpRoot.php` file. Be sure to check them out to see if they can be altered to fit your project a little better.

### Debugger
This library has a very small debug function that will allow you to output some request information and what was returned. This can be turned on by setting:

```php
const DEBUGGER = true;
```
The debugger can also log to a file if you provide a file for it to write to using `const DEBUGGER_LOG_FILE` like this:

```php
const DEBUGGER_LOG_FILE = 'path/to/some/file.php';
```

By default this option is set to `false`.

### SSL Verify
`CURLOPT_SSL_VERIFYPEER` can be disabled by setting:

```php
 const VERIFY_SSL = false;
````

By default this option is set to `true`.

### Request Headers
To include the returned headers in the output from this library set:

```php
const HEADERS = true;
```

By default this option is set to `false`.


\*\*Please watch for updates, and feel free to Fork or Pull Request. Check out the Wiki for a little more info on contributing.





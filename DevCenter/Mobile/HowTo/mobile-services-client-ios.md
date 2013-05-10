<properties linkid="mobile-services-how-to-iOS-client" urlDisplayName="iOS Client Library" pageTitle="How to use the iOS client library - Windows Azure Mobile Services feature guide" metaKeywords="Windows Azure Mobile Services, Mobile Service iOS client library, iOS client library" writer="glenga" metaDescription="Learn how to use the iOS client library for Windows Azure Mobile Services." metaCanonical="" disqusComments="1" umbracoNaviHide="0" />
<div chunk="../chunks/article-left-menu-iOS.md"></div>
# How to use the iOS client library for Mobile Services
<div class="dev-center-tutorial-selector"> 
  <a href="/en-us/develop/mobile/how-to-guides/how-to-dotnet-client" title=".NET Framework">.NET Framework</a>
  	<a href="/en-us/develop/mobile/how-to-guides/how-to-js-client" title="JavaScript">JavaScript</a> 
	<a href="/en-us/develop/mobile/how-to-guides/how-to-ios-client" title="Objective-C">Objective-C</a> 
	<a href="/en-us/develop/mobile/how-to-guides/how-to-android-client" title="Java">Java</a>
</div>	

This guide shows you how to perform common scenarios using the iOS client for Windows Azure Mobile Services. The samples are written in objective-C and require the [Mobile Services SDK].  This tutorial also requires the [iOS SDK]. The scenarios covered include querying for data; inserting, updating, and deleting data, authenticating users, handling errors, and uploading BLOB data. If you are new to Mobile Services, you should consider first completing the [Mobile Services quickstart][Get started with Mobile Services]. The quickstart tutorial helps you configure your account and create your first mobile service.

## Table of Contents

- [What is Mobile Services][]
- [Concepts][]
- [Setup and Prerequisites][]
- [How to: Create the Mobile Services client][]
- [How to: Create a table reference][]
- [How to: Query data from a mobile service][]
	- [Filter returned data]
    - [Sort returned data]
	- [Return data in pages]
	- [Select specific columns]
- [How to: Insert data into a mobile service]
- [How to: Modify data in a mobile service]
- [How to: Bind data to the user interface]
- [How to: Authenticate users]
	- [Cache authentication tokens]
- [How to: Handle errors]
- [How to: Design unit tests]
- [How to: Customize the client]
	- [Customize request headers]
	- [Customize data type serialization]
- [Next steps][]

<div chunk="../chunks/mobile-services-concepts.md" />

##<a name="Setup"></a>Setup and Prerequisites</h2>

This guide assumes that you have created a mobile service with a table.  For more information see [Create a table](http://msdn.microsoft.com/en-us/library/windowsazure/jj193162.aspx). In the code used in this topic, we assume the table is named *ToDoItem*, and that it has the following columns:

<ul>
<li>id</li>
<li>text</li>
<li>complete</li>
<li>duration</li>
</ul>

If you are creating your iOS application for the first time, make sure to add the "WindowsAzureMobileServices.framework" in your application's **Link Binary With Libraries** setting.

In addition, don't forget to add the following reference in the appropriate files or in your applications pch file.

	#import <WindowsAzureMobileServices/WindowsAzureMobileServices.h>

## <a name="create-client"></a>How to: Create the Mobile Services client

The following code creates the mobile service client object that is used to access your mobile service. 

	MSClient *client = [MSClient clientWithApplicationURLString:@"MobileServiceUrl" applicationKey:@"AppKey"]
	
In the code above, replace `MobileServiceUrl` and `AppKey` with the mobile service URL and application key of your mobile service. Both of these are available on the Windows Azure Management Portal, by selecting your mobile service and then clicking on "Dashboard".

If you have a NSURL object, you can use that to create your client as well.

	MSClient *client = [MSClient clientWithApplicationURL:(NSURL *)url
								 applicationKey:(NSString *)string];

## <a name="table-reference"></a>How to: Create a table reference</h2>

Before you can access data from your mobile service, you must get a reference to the table you want to query, update, or delete items from.

	MSTable *table = [client tableWithName:@"ToDoItem"]; // replace "ToDoItem" with the name of your table.

<h2><a name="querying"></a>How to: Query data from a mobile service</h2>

Once you have a MSTable object you can then create your query.  The following simple query gets all the items in our ToDoItem table.

	[table readWithCompletion:^(NSArray *items, NSInteger totalCount, NSError *error) {
		if(error) {
			NSLog(@"ERROR %@", error);
		} else {
			for(NSDictionary *item in results) {
				NSLog(@"Todo Item: %@", [item objectForKey:@"text"]);
			}
		}
	}];

Note that in this case we simply write the text of the task to the log.

The following parameters are available to you in the callback:
* *items*: An NSArray of the records that matched your query
* *totalCount*: is set to -1 unless your query asks for the total count (not just those returned in the query)
* *error*: the error that occurred, or nil

### <a name="filtering"></a>How to: Filter returned data

When you want to filter your results, you have a number of options available to you. 

The most common case is to use an NSPredicate to filter the results.

	[table readWithPredicate:(NSPredicate *)predicate completion:(MSReadQueryBlock)completion];
	
The following predicate returns only the incomplete items in our ToDoItem table:

	NSPredicate *predicate = [NSPredicate predicateWithFormat:@"complete == NO"];
	[table readWithPredicate:predicate completion:^(NSArray *items, NSInteger totalCount, NSError *error) {
		//loop through our results
	}];
	
A single record can be retrieved by using its Id.

	[table readWithId:[NSNumber numberWithInt:1] completion:^(NSDictionary *item, NSError *error) {
		//your code here
	}];

Note that in this case the callback parameters are slightly different.  Instead of getting an array of results and an optional count, you instead just get the one record back.

### <a name="query-object"></a>How to: Use MSQuery

Often you will need to do more than just specify a filter on your query. Use the MSQuery object when you need to change the sort order on your results or limit the number of data records you get back. The following shows how to create an MSQuery object instance:

	MSQuery *query = [table query];	
	MSQuery *query = [table queryWithPredicate:(NSPredicate *)predicate];

The MSQuery object enables you to control the following query behaviors:

* Specify the order results are returned.
* Limit which fields are returned.
* Limit how manu records are returned.
* Specify whether to include the total count in the response.
* Specify custom query string parameters in the request.

Once the query is configured, it is executed by calling the readWithCompletion function.

#### <a name="sorting"></a>Using MSQuery to sort returned data

The following functions are used to specify the fields to sort by and the direction:
	-(void) orderByAscending:(NSString *)field
	-(void) orderByDescending:(NSString *)field
	
This query sorts the results first by duration and then by whether the task is complete:

	[query orderByAscending(@"duration")];
	[query orderByAscending(@"complete")];
	[query readWithCompletion:(NSArray *items, NSInteger totalCount, NSError *error) {
		//code to parse results here
	}];	

#### <a name="paging"></a>Using MSQuery to get pages of data

Mobile Services limits the amount of records that are returned in a single response. However, to further reduce the number of records displayed to your users you must implement a paging system.  Paging is performed by using the following three properties of the MSQueery object:

	BOOL includeTotalCount
	NSInteger fetchLimit
	NSInteger fetchOffset

In the following example, a simple function requests 20 records from the server and then appends them to the local collection of all loaded record:

	- (bool) loadResults() {
		MSQuery *query = [self.table query];

		query.includeTotalCount = YES;
		query.fetchLimit = 20;
		query.fetchOffset = self.loadedItems.count;
		[query readWithCompletion:(NSArray *items, NSInteger totalCount, NSError *error) {
			if(!error) {
				//add the items to our local copy
				[self.loadedItems addObjectsFromArray:items];		

				//set a flag to keep track if there are any additional records we need to load
				self.moreResults = (self.loadedItems.count < totalCount);
			}
		}];
	}

#### <a name="selecting"></a>Using MSQuery to limit the fields returned

To limit which field are returned from your query, simply specify the names of the fields you want in the selectFields property. The following example returens only the text and completed fields:

	query.selectFields = @["text", @"completed"];

#### <a name="parameters"></a>Using MSQuery to specify additional querystring parameters

It's also possible to include additional querystring parameters in the request to the server. These parameters might be required by your server side scripts. For more information, see [How to: access custom parameters](http://www.windowsazure.com/en-us/develop/mobile/how-to-guides/work-with-server-scripts/#access-headers)

	query.parameters = @{
		@"myKey1" : @"value1",
		@"myKey2" : @"value2",
	};

These parameters are appended to the resulting request as part of the querystring.  In this example they would appear at the end as `myKey1=value1&myKey2=value2`

## <a name="inserting"></a><span class="short-header">Inserting data</span>How to: Insert data into a mobile service</h2>

To insert a new row into the table, you create a new NSDictionary object and pass that to the insert function. The following code inserts a new todo item into the table:

	NSDictionary *newItem = @{"text": "my new item", @"complete" : @NO};
	[table insert:newItem completion:^(NSDictionary *result, NSError *error) {
		//result will contain the new item that was inserted
		//depending on your server scripts it may have additional or modified data compared
		//to what was passed to the server
	}];	

Note: When you are insert an item, the insert failswhen you manually set 'id' in the dictionary object.

Also, when dynamic schema is enabled, Windows Azure Mobile Services automatically generates new columns based on the object in the insert or update request. For more information, see [Dynamic schema](http://go.microsoft.com/fwlink/?LinkId=296271).

## <a name="modifying"></a><span class="short-header">Modifying data</span>How to: Modify data in a mobile service</h2>

Update an existing object by modfiying an item returned from a previous query and then calling the update function.

	NSMutableDictionary *item = [self.results.item objectAtIndex:0];
	[item setObject:@YES forKey:"complete"];
	[table update:item completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];

When making updates, you only need to supply the field being update, along with the row id, as in the following example:

	[table update:@{"id" : 1, "Complete": Yes} completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];
	
	
To delete an item from the table, simply pass the item to the delete method.

	[table delete:item completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];

You can also just delete a record using its id directly.

	[table deleteWithId:[NSNumber numberWithInt:1] completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];	

The 'id' attribute must be set when making updates and deletes.

## <a name="authentication"></a>How to: Authenticate users</h2>

Mobile Services supports the following existing identity providers that you can use to authenticate users:

- Facebook
- Google 
- Microsoft Account
- Twitter

You can set permissions on tables to restrict access for specific operations to only authenticated users. You can also use the ID of an authenticated user to modify requests. For more information, see [Get started with authentication].

### Standard Login Workflow

Here is an example of how to login using FaceBook. This code could be called in your controller's ViewDidLoad or manually triggered from a UIButton. This will display a standard UI for logging into the identity provider.

	[client loginWithProvider:@"MicrosoftAccount" controller:self animated:YES
		completion:^(MSUser *user, NSError *error) {
			NSString *msg;
			if(error) {
				msg = [@"An error occured: " stringByAppendingString:error.description];
			} else {
				msg = [@"You are logged in as " stringByAppendingString:user.userId];
			}
		
			UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Login" 
								  message:msg 
								  delegate:nil 
								  cancelButtonTitle:@"OK" 
								  otherButtonTitles: nil];
			[alert show];
	}];	

Note: if you are using an identity provider other than Facebook, change the value passed to the login method above to one of the following: microsoftaccount, twitter, or google.

You can also get a reference to the MSLoginController and display it yourself using:

	-(MSLoginController *)loginViewControllerWithProvider:(NSString *)provider completion:(MSClientLoginBlock)completion;

### Using single-sign-on

When you already have an authenticated user you may want to instead log them in by providing the appropriate authentication token.

In this example, we will use the [Live SDK](http://msdn.microsoft.com/en-us/library/live/hh826532.aspx), which supports single-sign-on for iOS apps. In the most simplified form, we can use the cient flow as shown in this snippet. This code assumes that you have previously created a LiveConnectClient named liveClient in the controller and the user had logged in.
	
	[client loginWithProvider:@"microsoftaccount" 
		token:@{@"authenticationToken" : self.liveClient.session.authenticationToken}
		completion:^(MSUser *user, NSError *error) {
			self.loggedInUser = user;
			NSString *msg = [@"You are logged in as " stringByAppendingString:user.userId];
			UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Login" 
				message:msg 
				delegate:nil 
				cancelButtonTitle:@"OK" 
				otherButtonTitles: nil];
			[alert show];
	}];

? todo: Differences with the other SDKs

###<a name="caching-tokens"></a>How to: Cache authentication tokens

To prevent users from having to authenticate everytime they use run your application, you can cache the current user identity after they log in. You can then use this information to create the user directly and bypass the login process.  To do this you must store the User ID and authentication token locally. 

With a cached token, a user will not have to login again until the token expires. When a user tries to login with an expired token, a 401 unauthorized response is returned. At this point, the user must then log in to obtain a new token. You can use filters to avoid having to write code that handles expired tokens whenever your app that calls your mobile service.  Filters allow you to intercept calls to and responses from your mobile service. The code in the filter tests the response for a 401, triggers the login process if the token is expired, and then retries the request that generated the 401.

In the following example, we are caching this information securely in the [KeyChain](https://developer.apple.com/library/mac/#documentation/security/Conceptual/keychainServConcepts/02concepts/concepts.html#//apple_ref/doc/uid/TP30000897-CH204-TP9)

	- (NSMutableDictionary *) createKeyChainQueryWithClient:(MSClient *)client andIsSearch:(bool)isSearch
	{
		NSMutableDictionary *query = [[NSMutableDictionary alloc] init];
		[query setObject:(__bridge id)kSecClassInternetPassword forKey:(__bridge id)(kSecClass)];
		[query setObject:client.applicationURL.absoluteString forKey:(__bridge id)(kSecAttrServer)];
    	
		if(isSearch) {
			// Use the proper search constants, return only the attributes of the first match.
			[query setObject:(__bridge id)kSecMatchLimitOne forKey:(__bridge id)kSecMatchLimit];
			[query setObject:(id)kCFBooleanTrue forKey:(__bridge id)kSecReturnAttributes];
			[query setObject:(id)kCFBooleanTrue forKey:(__bridge id)kSecReturnData];
		}
    
		return query;
	}

	- (IBAction)loginUser:(id)sender {
		NSMutableDictionary *query = [self createKeyChainQueryWithClient:self.todoService.client andIsSearch:YES];
		CFDictionaryRef cfresult;

		OSStatus status = SecItemCopyMatching((__bridge CFDictionaryRef)query, (CFTypeRef *)&cfresult);
		if (status == noErr) {
			NSDictionary * result = (__bridge_transfer NSDictionary*) cfresult;
			
			//create an MSUser object
			MSUser *user = [[MSUser alloc] initWithUserId:[result objectForKey:(__bridge id)(kSecAttrAccount)]];
			NSData *data = [result objectForKey:(__bridge id)(kSecValueData)];
			user.mobileServiceAuthenticationToken = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
			[self.todoService.client setCurrentUser:user];
			
		} else if (status == errSecItemNotFound) {
			//we need to log the user in
			[self.todoService.client loginWithProvider:@"MicrosoftAccount" controller:self animated:YES
				completion:^(MSUser *user, NSError *error) {
					NSString *msg = [@"You are logged in as " stringByAppendingString:user.userId];
					UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Login" 
											message:msg delegate:nil 
											cancelButtonTitle:@"OK" 
											otherButtonTitles: nil];
					[alert show];
                
					//save the user id and token to the KeyChain
					NSMutableDictionary *newItem = [self createKeyChainQueryWithClient:self.todoService.client 
															andIsSearch:NO];
					[newItem setObject:user.userId forKey:(__bridge id)kSecAttrAccount];
					[newItem setObject:[user.mobileServiceAuthenticationToken dataUsingEncoding:NSUTF8StringEncoding] 
                                                    forKey:(__bridge id)kSecValueData];
                 
					OSStatus status = SecItemAdd((__bridge CFDictionaryRef)newItem, NULL);
					if(status != errSecSuccess) {
						//handle error as needed
						NSAssert(NO, @"Error caching password.");
					}
			}];
		}	

Note: This data is sensitive, and it should be stored encrypted for safety in case the phone gets stolen.


Todo: Show code for handling an expired token: Service filter onto the client. For details you can see [Handling Expired Tokens](http://www.thejoyofcode.com/Handling_expired_tokens_in_your_application_Day_11_.aspx).


<h2><a name="errors"></a><span class="short-header">Error handling</span>How to: Handle errors</h2>

_This section shows how to perform property error handling._

<h2><a name="#unit-testing"></a><span class="short-header">Designing tests</span>How to: Design unit tests</h2>

_(Optional) This section shows how to write unit test when using the client library (info from Yavor)._

<h2><a name="#customizing"></a><span class="short-header">Customizing the client</span>How to: Customize the client</h2>

_(Optional) This section shows how to send customize client behaviors._

###<a name="custom-headers"></a>How to: Customize request headers

_(Optional) This section shows how to send custom request headers._

For more information see, New topic about processing headers in the server-side.

###<a name="custom-serialization"></a>How to: Customize serialization

_(Optional) This section shows how to use attributes to customize how data types are serialized._

For more information see, New topic about processing headers in the server-side.

## <a name="next-steps"></a>Next steps

<!-- Anchors. -->

[What is Mobile Services]: #what-is
[Concepts]: #concepts
[Setup and Prerequisites]: #Setup
[How to: Create the Mobile Services client]: #create-client
[How to: Create a table reference]: #table-reference
[How to: Query data from a mobile service]: #querying
[Filter returned data]: #filtering
[Sort returned data]: #sorting
[Return data in pages]: #paging
[Select specific columns]: #selecting
[How to: Bind data to the user interface]: #binding
[How to: Insert data into a mobile service]: #inserting
[How to: Modify data in a mobile service]: #modifying
[How to: Authenticate users]: #authentication
[Cache authentication tokens]: #caching-tokens
[How to: Upload images and large files]: #blobs
[How to: Handle errors]: #errors
[How to: Design unit tests]: #unit-testing 
[How to: Customize the client]: #customizing
[Customize request headers]: #custom-headers
[Customize data type serialization]: #custom-serialization
[Next Steps]: #next-steps

<!-- Images. -->

<!-- URLs. -->
[Get started with Mobile Services]: ../tutorials/mobile-services-get-started-iOS.md
[Mobile Services SDK]: https://go.microsoft.com/fwLink/p/?LinkID=266533
[Get started with authentication]: ./mobile-services-get-started-with-users-iOS.md
[iOS SDK]: https://developer.apple.com/xcode

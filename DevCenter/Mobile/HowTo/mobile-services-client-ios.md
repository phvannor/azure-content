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

If you are creating your iOS application for the first time, make sure to add the "WindowsAzureMobileServices.framework" in your application's Link Binary With Libraries setting.

In addition, don't forget to add the following reference in the appropriate files or in your applications pch file.

	#import <WindowsAzureMobileServices/WindowsAzureMobileServices.h>

todo: where should this go:  When dynamic schema is enabled, Windows Azure Mobile Services automatically generates new columns based on the object in the insert or update request. For more information, see [Dynamic schema](http://go.microsoft.com/fwlink/?LinkId=296271).

## <a name="create-client"></a>How to: Create the Mobile Services client

The following code creates the mobile service client object that is used to access your mobile service. 

	MSClient *client = [MSClient clientWithApplicationURLString:@"MobileServiceUrl" applicationKey:@"AppKey"]
	
In the code above, replace `MobileServiceUrl` and `AppKey` with the mobile service URL and application key of your mobile service. Both of these are available on the Windows Azure Management Portal, by selecting your mobile service and then clicking on "Dashboard".

If desired, you can also create your client using an NSURL object.

	MSClient *client = [MSClient clientWithApplicationURL:(NSURL *)url
								 applicationKey:(NSString *)string];

## <a name="table-reference"></a>How to: Create a table reference</h2>

The first thing you need is to get a reference to the table you want to query, update, or delete items from.

	MSTable *table = [client tableWithName:@"ToDoItem"]; // replace "ToDoItem" with the name of your table.

<h2><a name="querying"></a>How to: Query data from a mobile service</h2>

Once you have a MSTable object you can then create your query.  To start, lets do a simple query to get all the items in our todo table.  In this case, we will just write the text of the task to the log.

	[table readWithCompletion:^(NSArray *items, NSInteger totalCount, NSError *error) {
		if(error) {
			NSLog(@"ERROR %@", error);
		} else {
			for(NSDictionary *item in results) {
				NSLog(@"Todo Item: %@", [item objectForKey:@"text"]);
			}
		}
	}];

The parameters available to you in the callback are:
* items: An NSArray of the records that matched your query
* totalCount: is set to -1 unless your query asks for the total count (not just those returned in the query)
* error: the error that occurred, or nil

### <a name="filtering"></a>How to: Filter returned data

When you want to filter your results, you have a number of options available to you. To get a specific record from the table you can do

	[table readWithId:[NSNumber numberWithInt:1] completion:^(NSDictionary *item, NSError *error) {
		//your code here
	}];

Note that in this case the callback parameters are slightly different.  Instead of getting an array of results and an optional count, you instead just get the one record back.

When you want to filter your results instead, you can use

	[table readWithPredicate:(NSPredicate *)predicate completion:(MSReadQueryBlock)completion];
	
So if we wanted to get only the items in our todo table that were incomplete we could do

	NSPredicate *predicate = [NSPredicate predicateWithFormat:@"complete == NO"];
	[table readWithPredicate:predicate completion:^(NSArray *items, NSInteger totalCount, NSError *error) {
		//loop through our results
	}];


### <a name="query-object"></a>How to: Use MSQuery

Often you will need to do more than just specify the filter on your query. When you want to change the sort order on your results, or limit the amount you get back, you will want to get a MSQuery object.

	MSQuery *query = [table query];	
	MSQuery *query = [table queryWithPredicate:(NSPredicate *)predicate;

The MSQuery object lets you
* Specify the order results are returned
* Limit which fields are returned
* Set how many results to return
* Whether you want to include the total count
* Specify your own query string parameters to send

Once you have finished modifying the query, call the readWithCompletion method

	[query readWithCompletion:(NSArray *items, NSInteger totalCount, NSError *error) {
		//code to parse results here
	}];

#### <a name="sorting"></a>Using MSQuery to sort returned data

To sort your data you specify the field you want to sort by and the direction using:
	-(void) orderByAscending:(NSString *)field
	-(void) orderByDescending:(NSString *)field
	
If we wanted to sort our to do list by duration and then whether it was complete we would do:

	[query orderByAscending(@"duration")];
	[query orderByAscending(@"complete")];
	[query readWithCompletion:(NSArray *items, NSInteger totalCount, NSError *error) {
		//code to parse results here
	}];	

#### <a name="paging"></a>Using MSQuery to get pages of data

Instead of showing all results at once, you typically need to set up a paging system.  This can be done by using the following three properties of the MSQuery object

	BOOL includeTotalCount
	NSInteger fetchLimit
	NSInteger fetchOffset

In this example, we have a simple function that grabs 20 records from the server and appends them to a local copy of all the records loaded.

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

To limit which field are returned from your query, simply specify the names of the fields you want in the selectFields property.

	query.selectFields = @["text", @"completed"];

#### <a name="parameters"></a>Using MSQuery to specify additional querystring parameters

Its also possible to send along additional querystring parameters to the server for your server side scripts to process.  [todo: link to appropriate information on this]

	query.parameters = @{
		@"myKey1" : @"value1",
		@"myKey2" : @"value2",
	};

These will be appended to the resulting request as part of the querystring.  In this example they would appear at the end as myKey1=value1&myKey2=value2

## <a name="inserting"></a><span class="short-header">Inserting data</span>How to: Insert data into a mobile service</h2>

To insert a new row into the table, you create a new NSDictionary object and pass that into your table object. Let's insert a new todo item into our table.

	NSDictionary *newItem = @{"text": "my new item", @"complete" : @NO};
	[table insert:newItem completion:^(NSDictionary *result, NSError *error) {
		//result will contain the new item that was inserted
		//depending on your server scripts it may have additional or modified data compared
		//to what was passed to the server
	}];	

When you are inserting an item, it will fail if you manually set 'id' in the dictionary object.

## <a name="modifying"></a><span class="short-header">Modifying data</span>How to: Modify data in a mobile service</h2>

To update an existing object, you can modify the item directly from a previous query and pass it back to the table object.

	NSMutableDictionary *item = [self.results.item objectAtIndex:0];
	[item setObject:@YES forKey:"complete"];
	[table update:item completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];

? todo: what happens if you leave a field out of the dictionary, is it cleared

To delete that item instead you can call

	[table delete:item completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];

or you can delete a record using its id directly if you don't have the item.

	[table deleteWithId:[NSNumber numberWithInt:1] completion:^(NSDictionary *item, NSError *error) {
		//handle errors or any additional logic as needed
	}];	

For both update and delete, when using an item the 'id' attribute must be set.

## <a name="authentication"></a><span class="short-header">Authentication</span>How to: Authenticate users</h2>

Mobile Services supports the following existing identity providers that you can use to authenticate users:

- Facebook
- Google 
- Microsoft Account
- Twitter

You can set permissions on tables to restrict access for specific operations to only authenticated users. You can also use the ID of an authenticated user to modify requests. For more information, see [Get started with authentication].

Once you have chosen your preferred method for authentication, and set it up for your mobile services you can add the following code to start the login process for a user

	[client loginWithProvider:@"MicrosoftAccount" controller:self animated:YES 
			completion:^(MSUser *user, NSError *error) {
				
	}];
	
You can use Google, Facebook, or Twitter instead of MicrosoftAccount.  For the controller, specify the controller that will display the login UI.

When this is called for the first time, the user will be presented with the appropriate login screen asking them for their user id and password.



###<a name="caching-tokens"></a>How to: Cache authentication tokens

This section shows how to cache an authentication token. Do this to prevent users from having to authenticate again if app is "hibernated" while the token is still vaid.

To do this you must store the User ID and authentication token locally on the device. The next time the app starts, you check the cache, and if these values are present, you can skip the login procedure and rehydrate the client with this data. However this data is sensitive, and it should be stored encrypted for safety in case the phone gets stolen.

So what happens if your token expires? In this case, when you try to use it to connect, you will get a 401 unauthorized response. The user must then log in to obtain new tokens. You can avoid having to write code to handle this in every place in your app that calls Mobile Servides by using filters, which allow you to intercept calls to and responses from Mobile Services. The filter code will then test the response for a 401, trigger the login process if needed, and then resume the request that generated the 401.

	//fix to be secure storage
	if(self.user) {
		[client loginWithProvider:@"MicrosoftAccount" token:self.user.mobileServiceAuthenticationToken 
			completion:^(MSUser *user, NSError *error) {
		
		}];
	} else {
		[client loginWithProvider:@"MicrosoftAccount" controller:self animated:YES 
				completion:^(MSUser *user, NSError *error) {
					self.user = user;				
		}];	
	}
	

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
[Mobile Services SDK]: http://go.microsoft.com/fwlink/?LinkId=257545
[Get started with authentication]: ./mobile-services-get-started-with-users-iOS.md

---
layout: post
title: "Migrating to the new Email Search API"
description: "If you use the Search API to map users by an email address, it's time to move to the new Email Search API"
date: 2017-10-24 08:30
category: Technical Guide, Email Search API
author:
  name: "Chris Moyer"
  url: "https://blog.coredumped.org
  mail: "kopertop@gmail.com"
  avatar: "https://en.gravatar.com/avatar/54d15d5d1700546270365e272098ca48?s=200"
tags:
- Email Search
- API
- Authentication Workflow
- New Features
---

One very common use case for the Auth0 Search API is to identify users by email address, merging accounts together, or linking existing accounts based on a shared email address. In the case of my company, Cannabiz Media, we also used this API to identify new users and create accounts when a user signed up, de-duplicating by email address to prevent users from having multiple accounts.

## Introduction to the Search API

The search API is a generic API for searching Users, including with the ability to search for any generic metadata associated with a user's ```app_metadata``` or ```user_metadata```. This means if you had something like:

```json
"app_metadata": {
  "plan": "trial"
}
```

you could easily search for any users on the Trial plan by doing a search like:

```
app_metdata.plan:Trial
```

That's great for general searches, but it means that Auth0 has to store a lot of indexes, including a lot for things people never search for. In fact, most of the uses for the search API were just for looking up users by email. If you've ever set up a Search system, you know how ineficient it is to have unused indexes, and what kind of impact that has to search.

### Why using the Search API for authentication workflow is bad

The search API isn't designed for high availability. Over the past week, I've noticed at least 3 occasions when it had issues (either complete outages, or just a partial outage). Your system is only as strong as it's weakest link, so if your login requires access to a system that's frequently unavailable, your entire system is effectively broken.

### Use-cases for mapping by email

Auth0 allows you to have multiple identities (authentication mechanisms) for the same account. For example, you could set up a Database (username/password) account that also is connected to your Gmail login, no matter which login you use, you'll be connected to the same account.

In fact, the recommended rule to [Link accounts by email address](https://github.com/auth0/rules/blob/master/rules/link-users-by-email.md) used the Search API to do this.

```javascript
function (user, context, callback) {
   var request = require('request@2.56.0');

   var userApiUrl = auth0.baseUrl + '/users';

   console.log('Checking for user', user.email, context.connectionStrategy);
   request({
      url: userApiUrl,
      headers: {
         Authorization: 'Bearer ' + auth0.accessToken,
      },
      qs: {
         search_engine: 'v2',
         q: 'email.raw:"' + user.email + '" -user_id:"' + user.user_id + '"',
      },
   },function(err, response, body) {
     // Don't completely break authentication just because
     // the Auth0 Search API is broken
      if (err || response.statusCode !== 200){
        console.error('ERROR Checking for merge accounts', err);
        return callback(null, user, context);
      }

      var data = JSON.parse(body);
      console.log('Got', data.length, 'matches');
      data.forEach(function(targetUser){
         var aryTmp = targetUser.user_id.split('|');
         var provider = aryTmp[0];
         var user_id = aryTmp[1];
         // Don't allow merging in other auth0 users
         if(provider !== 'auth0'){
           console.log('Merge', targetUser.email, provider, user_id);
           request.post({
              url: userApiUrl + '/' + user.user_id + '/identities',
              headers: {
                 Authorization: 'Bearer ' + auth0.accessToken
              },
              json: { provider: provider, user_id: user_id }
           }, function(err, response, body) {
               if (response.statusCode >= 400) {
                  console.log('Error linking account:',response.statusMessage);
               }
           });
         }
      });
     
      // Return the OK response for this user record
      callback(null, user, context);
   });
}
```

It's a common request, a user logs in with their Google account and they get migrated right into their normal account.

### The Search API isn't designed for maximum availability

The search system isn't designed for this kind of usage. It's designed for *flexibility*, not *availability*. Since most of the searches are just for looking up by email, Auth0 came up with a better solution. Enter the Email Lookup API.

## Introduction to the Email Lookup API

The Email Lookup API doesn't use the search system, with it's thousands of different indexes. Instead, it uses a very simple index on the primary data store, the same DB that has had 99.99% uptime over the past 6 months. Using this API in your authentication workflow is completely safe.

### What code to replace, and how to migrate from the Search API

In order to make the migration, there's only a few lines in the original rule we need to replace. Instead of hitting ```/users```, we need to hit ```/users-by-email```:

```javascript
   var userApiUrl = auth0.baseUrl + '/users';
   var userSearchApiUrl = auth0.baseUrl + '/users-by-email';
   request({
    url: userSearchApiUrl,
    headers: {
      Authorization: 'Bearer ' + auth0.accessToken
    },
    qs: {
      email: user.email
    }
  },
```

Unlike with the search API, this API doesn't allow additional filters such as excluding the current user, so we also need to add a filter to our data:

```javascript
      // Ignore the current user, if present
      data = data.filter(function(u) {
        return (u.user_id !== user.user_id);
      });
```

Other than that, the data returned is identical so there's no problem with the remainder of our rule.

### Limitations, things that it can not be used for

As mentioned briefly above, there's no additional filters allowed via the ```users-by-email``` API, so any additional filtering needs to be done client-side. You also can't search for anything in the ```app_metadata```, so if you rely on rules to match things like by a ```customer_id``` or other ID outside of the email address, you'll still need to rely on the Search API. However, keep in mind that anything that relies on the Search API will not be available 99.99% of the time.

## How Cannabiz Media uses the new Email Lookup API

Of course, linking users by email address is the most common use-case and essentially why this new API endpoint was created. However, there are other reasons to need to look up a user by email address. At my company, [Cannabiz Media](https://cannabiz.media), we use WooCommerce as our Storefront, which automatically creates user accounts in Auth0 when a user purchases a new subscription. A WooCommerce Webhook triggers an AWS Lambda function via API Gateway whenever a new subscription is purchased, updated, or cancelled. The payload looks similar to this:

```json
{
  "billing": {
    "id": 1,
    "name": "Chris Moyer",
    "email": "kopertop@gmail.com",
    "username": "kopertop"
  },
  "primary_subscription": {
    "id": "1",
    "plan": "startup",
    "price": 3600,
    "period": "yearly",
    "status": "Active"
    "start-date": "2017-10-25T00:00:00"
  }
}
```

As I mentioned, this webhook would be fired if the subscription was added, updated, or cancelled, so we couldn't assume that just because the webhook fired this was a brand-new user. Obviously there's no Auth0 ID there, so we used the ```billing.email``` to see if there already was a user for this account, or if we needed to create a new one.

### How we used the Search API

The old Search API was the only solution at the time, so we implemented the webhook using a lookup method like the following:

```javascript

/**
 * Create or update an account
 */
export async function createAccount(billing, app_metadata, account_details) {
   const auth0Client = await getAuth0ClientPromise();
   const user: UserProfile = await getUserByEmail(account_details.email);

   if (user) {
      console.log('UPDATE user', {
         account_details,
         app_metadata,
         billing,
      });
      ...
   } else {
      console.log('CREATE user', {
         account_details,
         app_metadata,
         billing,
      });
      ...
   }
}
/**
 * Check for an existing user by Email
 */
export async function getUserByEmail(email) {
   let matchedUser;

   const auth0Client = await getAuth0ClientPromise();
   const users = await auth0Client.getUsers({ q: '"' + email + '"', search_engine: 'v2' });

   if(users && users.length > 0){
      users.forEach((user) => {
         if (!matchedUser && user.email === email) {
            matchedUser = user;
         }
      });
   }
   if(!matchedUser){
      // If we couldn't find the user using Auth0, check DynamoDB
      const ddbResponse = await ddb.query({
         TableName: 'UserProfiles',
         IndexName: 'email-index',
         KeyConditions: {
            email: {
               ComparisonOperator: 'EQ',
               AttributeValueList: [ email ],
            },
         },
      }).promise();
      if(ddbResponse && ddbResponse.Items && ddbResponse.Items.length > 0){
         ddbResponse.Items.forEach((item) => {
            if(!matchedUser && item && item.email === email){
               console.log('DDB Record found');
               matchedUser = item;
            }
         });
      }
   }

   return matchedUser;
}
```

Notice that we often had issues with the search API, so instead of just relying on the fact that any user that wasn't found just didn't exist in Auth0, we also stored all users in a DynamoDB Index just to be sure, and checked that as a fallback.

### Making the Switch to the Users-By-Email API

The search API had issues, and one of the problems we experienced was not being able to add new subscribers, or modify subscribers, when the Search API was down. We had users subscribing while the Search API was down, and we were getting angry calls from our customers wondering why they couldn't log in yet. Worst of all, before we double-checked in DynamoDB, we ended up with *duplicate accounts*, and sometimes sent our customers multiple welcome emails.

Each user we create has an Export Quota. Giving users multiple accounts wasn't only confusing, it also could end up with them abusing the system, giving them twice (or more) the amount of exports they should have had.

Fortunately, we had one function which was used to fetch users by an email address, so we just had to make the code change in one spot:

```javascript
/**
 * Check for an existing user by Email
 */
export async function getUserByEmail(email) {
   let matchedUser;

   const auth0Client = await getAuth0ClientPromise();
   const users = await auth0Client.getUsersByEmail(email);
   ...
```

**DEVELOPER'S NOTE** As of the time of this blog post, you need to use [my Fork of the Node.js Auth0 API](https://github.com/kopertop/node-auth0). Hopefully this is merged in before this post is pushed out.


### Performance/Reliability improvements

The obvious improvement here is that the ```/users-by-email``` system is much more reliable, but the unexpected side-effect is that user account creation is also a bit quicker now. The main reason? We don't have to have the duplicate checking of DynamoDB, since we're certain that the ```/users-by-email``` endpoint will return all valid users. Additionally, we don't have to store those users in DynamoDB, saving us a bit of money on our AWS bill, but also complexity in our code.

Before this API, we were contemplating a way to migrate off of Auth0 entirely. Now we're sticking with Auth0, which will save us a lot of uneccessary development time since Auth0's widgets and APIs perform a lot of tasks we wouldn't want to have to handle ourselves.


## Conclusion

Auth0's Search API uses ElasticSearch. Anyone who's ever dealt with ElasticSearch knows how complicated it can be to get it right. What makes Auth0's situation even more complex is that they're using it to host *customers* data, which can be any random format, not something that adhears to a strict format. Auth0 knows this, and they've come up with a good solution for 90% of the use-cases for search.

The only other place at Cannabiz we use search is on the management console, to lookup users accounts when they have a complaint. It's not a mission-critical use-case, so if there's temporary issues in the search API, it no longer will mean we can't create new users, or link user accounts by email addresses. We had already added work-around logic to prevent a search API outage from blocking all user authentication, but now we can also rest assured that it won't impact *anything mission critical*.


### Email Search is more limitted, but fits the general use case

Email Searching doesn't solve all of the problems. Auth0 will still need to work on making the Search API more resiliant and reliable for developers, but it's no longer a critical issue. The issue identified fits our use case, and probably will fit yours too. It might take a little bit more work to get into your workflow, but it's well worth it to ensure users have a good experience.

For new developers just starting out, the new API is actually simpler. Instead of having to craft a special query, you just hit an endpoint.


### There are more use-cases than just the Account-linking rule

As you can see from our use-case, looking up a user by email is useful to a lot more applications then just an Auth0 rule to merge accounts. In fact, we're currently using it to help us with account creation, and we're using it to link users from Auth0 to our Analytics platform. The new API should also be significantly less resource-intensive for Auth0, and with fewer people using the Search API for these frequent request, hopefully even the Search API will have fewer requests, which should make it less stressed and more reliable in the long run.

Yes, search still needs to be fixed, but this API should reduce the impact search has on your workflow.

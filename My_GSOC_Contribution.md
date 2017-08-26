# CloudDB

## Why CloudDB?

One of the interesting use cases for App
Inventor involves people wanting to build an app that works “offline”
when/where no internet access is available, let the user record data
and then upload this data when internet access becomes available. While developers of App Inventor apps have support for storing data locally and for storing data remotely, App Inventor does not allow developers to develop apps that work offline. The idea of this project was to come up with a database that would would work both offline and online. We call this database CloudDB.

## What is CloudDB?

CloudDB has two components - a local SQLite Database and a remote Redis server. When the device is offline all interactions with CloudDB take place with the local SQLite Database. When the device is online, data in the local SQLite database is synced with the remote Redis server. Moreover, when the device is online all interactions with CloudDB occur with the remote Redis server. Redis is a light weight data store where data is stored in the form of ke-value pairs. It supports multiple data structures ranging from strings, hashes, sets, sorted sets etc. Redis is extremely fast because it uses an in-memory dataset. More information about Redis can be found [here](https://redis.io/).

## CloudDB implementation on App Inventor.

### Designing the local cache.

A major component of CloudDB is the local cache. The local cache allows CloudDB to operate in both online and offline mode. The local cache is a SQLite Database with two tables -

1. *table1* for storing user's data
2. *table2* for storing user's authentication details.

*table1* has a *key*, *value*, *uploadFlag* i.e a flag to denote whether it has been uploaded to the remote database, and a *timestamp* to reflect the time it was captured. We will discuss about *table2* in the *Designing Authentication for CloudDB* section. Check [CloudDBCache.java](https://github.com/JoyMitra/appinventor-sources/blob/joy_dev/appinventor/components/src/com/google/appinventor/components/runtime/util/CloudDBCache.java) and [CloudDBCacheHelper.java](https://github.com/JoyMitra/appinventor-sources/blob/joy_dev/appinventor/components/src/com/google/appinventor/components/runtime/util/CloudDBCacheHelper.java) for more details about the structure of the cache.

When a client wants to read/write data from/in CloudDB, CloudDB checks whether the device is online or offline. If the device is offline then read/write happens from/to the local cache as a *key, value* pair. If the device is online then read/write happens from/to the remote Redis database. In App Inventor we use the [Jedis](https://github.com/xetorthio/jedis) library as an interface to Redis. Check [CloudDB.storeValue()](https://github.com/JoyMitra/appinventor-sources/blob/joy_dev/appinventor/components/src/com/google/appinventor/components/runtime/CloudDB.java) for an example of how CloudDB saves data.

### Designing the Sync Background Process.

An important aspect of having a database that works both online and offline is to find a way to keep the online and offline versions in sync. There are two ways of doing this.

1. allow the user to explicitly start a sync. Developers can set up CloudDB in a way that will require users of their apps to explicitly sync their local data with the remote database.

2. start a sync automatically. Developers can set up CloudDB such that users do not have to care about sync. Users' data will be saved in CloudDB irrespective of them being online or offline.

1 is trivial. 2 is a little complicated because it requires scheduling a background process that will be responsible for keeping the online and offline versions in sync. The main challenge with designing such a background process on Android is to determine when to start this process and when to stop it. It is not very hard to design a background process that is not aware of the device's resource constraints and ends up draining battery. Fortunately Android provides a few APIs to design long running background processes.

1. *AlarmManager*. The AlarManager API is available for all versions of Android. It allows developers to specify a time at which a background process should run. However, the API behavior is not consistent across versions of Android. It requires a lot of boiler plate code. Most importantly it does not recognize device state which means that efficiently scheduling a long running background process is the responsibility of the developer.

2. *JobScheduler*. The JobScheduler API was designed to help developers schedule long running background jobs at a particular time or periodically keeping device state in mind. However, this is only available for APIs 21 and above and a lot of App Inventor apps need to run on devices that run older versions of Android. Also, the JobScheduler API has a lot of boiler code not unlike the *AlarmManager* API.

3. *GCMNetworkManager*. The GCM NetworkManager API is similar to *JobScheduler* with support for older versions of Android up to 2.3. However, this API does not work on devices that do not have *Google Play Services* pre-installed and a large number of App Inventor developers develop apps for Amazon Android devices which do not have *Google Play Services*.

While support for scheduling long running background jobs efficiently exists in Android, none of them fit our requirements. To summarize we needed to design a background sync process that has the following requirements.

1. perform a sync periodically.

2. perform a sync only when device is online.

3. perform a sync intelligently keeping device state in mind.

4. sync does not have to be instantaneous. It can wait till the device finds a suitable time to perform the sync.

5. should work on older versions of Android up to SDK Level 9.

6. should not have dependencies on the user's end like pre-installed *Google Play Services*.

Fortunately, we found a utility library [Android-Job](https://github.com/evernote/android-job) that met all our requirements. This library uses *AlarmManager*, *JobScheduler*, *GCMNetworkManager* depending on the Android version. We integrated this library to run with AppInventor.

We use this library to specify that a sync should be scheduled to run as a background process periodically every 15 minutes, when the device is connected to the internet. The sync background process will connect to the Redis server via an *authentication token* (discussed later), select *key/value* pairs whose *uploadFlag* is set to 0 (pairs not uploaded to the server yet) from *table1* in the local SQLite Database and upload them to the Redis server. It then sets the *uploadFlag* to 1 for the *key/value* pairs that were uploaded. Check [SyncJob](https://github.com/JoyMitra/appinventor-sources/blob/joy_dev/appinventor/components/src/com/google/appinventor/components/runtime/util/SyncJob.java) for details of how the sync is performed.

### Designing authentication for CloudDB

CloudDB is powered by Redis. Therefore, in its basic form CloudDB is a key/value datastore. The key for some data that needs to be persisted is generated from a developer-provided string, the developer account name, and the project name provided by the developer. A malicious user with access to the Redis Server could easily generate am existing key and could get access to the data stored under that key. We addressed this issue by creating a unique namespace for every user of CloudDB.

We developed a filter on top of Redis in our server. This filter program  receives an *authentication token* which contains the hash of the unique user id and a digital signature which is a hash of the hashed unique user id. The digital signature is used to authenticate the user id. Once authentication is successful the hashed unique user id is prepended to the key of the key/value pair that needs to be persisted. This way a malicious user with access to the Redis server will not be able to access the key/value pairs of other users since in order to do that, the malicious user will need to re-produce the hash of the victim's unique user id which is very difficult because the hash of the victim's unique user id is produced by a secret key known only to the app inventor server.

The *authentication token* consists of two parts - a hash of the unique user id and a digital signature.

1. Every user of App Inventor has a unique user id (uuid) generated by the App Inventor server. This uuid is hashed with a secret key known only to the App Inventor Server.

2. The digital signature is created from the hashed uuid. The hashed uuid is hashed again with a secret key shared between the App Inventor server and the Redis server.

Both 1 and 2 are set in an instance of a [Google Protocol Buffer](https://developers.google.com/protocol-buffers/) object. Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data. The *Protocol buffer* object is Base58 encoded and set as a [designer property](http://explore.appinventor.mit.edu/support/component-properties) for the CloudDB component via RPC. When the CloudDB component is instantiated by the App Inventor system, this token is saved in *table2* of the local cache so that it can be used by the background sync process to connect to the Redis server. Check [CloudDBAuthServiceImpl.java](https://github.com/JoyMitra/appinventor-sources/blob/joy_dev/appinventor/appengine/src/com/google/appinventor/server/cloudDBAuth/CloudDBAuthServiceImpl.java) for more details about token generation.

## References

1. [A unified job library for Android](https://blog.evernote.com/tech/2015/10/26/unified-job-library-android/)

2. [MIT App Inventor Documentation](http://appinventor.mit.edu/appinventor-sources/#documentation)

3. [Redis](https://redis.io/)

4. [Google Protocol Buffers](https://developers.google.com/protocol-buffers/)

5. [Hashing in Android](https://developer.android.com/reference/javax/crypto/Mac.html)

6. [RPC in Google AppEngine](https://cloud.google.com/appengine/docs/standard/python/tools/protorpc/)


Note : The joy_dev branch consists of the CloudDB work so please fork this branch if you want to see CloudDB. 

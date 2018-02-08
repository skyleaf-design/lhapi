# LHAPI technical specification







## LHAPI web service

The LHAPI service is written in JavaScript, which runs on top of the Node runtime.



### File structure and purpose

    server ⇾ source code files for the LHAPI web service
      ◇ lhapi.js ⇾ main entry point for server-side code
      ↳ resource ⇾ files and folders to handle LHAPI routes
        ◇ affordance.js ⇾ imported by lhapi.js
        ◇ maker.js ⇾ handles maker route, forms query for all HTTP actions
        ◇ program.js ⇾ requires mysql connection from lhapi.js
      ↳ static_content ⇾ all files are served via HTTP, publicly
        ↳ maker_image ⇾ images of makers for their profiles
          ◇ 1.png ⇾ named with maker identifier
          ◇ 2.png
        ↳ program_screenshot ⇾ images of screenshots of uploaded programs
          ◇ 1.png ⇾ named with program identifier
          ◇ 2.png
        ↳ program_video ⇾ images of screenshots of uploaded programs
          ◇ 1.png ⇾ named with program identifier
          ◇ 2.png
      ↳ storage ⇾ code files that replate to migrating mysql to different schemas when upgraded
        ◇ current_state.json ⇾ current schema version
        ↳ upgrade_scripts ⇾ these upgrade one database schema version to another.
          ◇ mysql_001.sh 255 limit ⇾ zero-padded
          ◇ mysql_002.sh ⇾ enough digits to fill 8-bit int
        


### Architecture

All the LHAPI service really does is handle authentication, serve the static client, and return HTTP responses when the client asks it questions.  It hands off persistent storage responsibilities to MySQL, and screenshot/video generation to headless Chrome.

#### Application Logic
Content is served with Node's built-in `http` server.  Static content is served with `ecstatic`.  Routes are parsed with the `routes` npm package.  Authetication is done via JWT tokens, via `jwt-js` or similar library.  Unit tests and integration tests are done with `tape`, while end-to-end tests are done with a headless browser, such as PhantomJS or headless Chrome.

#### Caching... there is none.
Aside from MySQL query caching, no caching is performed inside the LHAPI service whatsoever.  The main item to cache would be the static client, but this can be cached via standard HTTP caching methods, since it is static content.

#### Scaling
The LHAPI service should run in a Node cluster, since we are using the separate MySQL process as our single source of truth.

The LHAPI service maintains no application state of its own, and therefore should run well in a standard Node cluster.

#### Persistent Storage
When a maker uploads a program, the LHAPI service immediately generates the following records:
* The LHAPI service immediately generates a `program` database record, and forms a query to insert it into the appropriate tables in MySQL
* The LHAPI services calls out to MySQL to store these records.
* After these records are in place, the program will be viewable at the URL `root_url/[program_uuid]`

#### Media Generation
When a maker uploads a program, the LHAPI service immediately generates the following entities:
* A placeholder screenshot image is immediately copied to `root_file_url/static_content/program_screenshot/[program_uuid].png`.
* A placeholder video is immediately copied to `root_file_url/static_content/program_video/[program_uuid].png`.
* The LHAPI service calls out to headless Chrome with a script, and the script tells Chrome to take screenshots and video of the running program, and overwrite the existing placeholders with these files.
* The placeholder and actual files are always publicly-viewable, because the entire `static_content` directory is served by the LHAPI service.



### HTTP Resources

When a program is requested for viewing, LHAPI delivers a client.  The client then requests content from LHAPI's REST API.  Other UI clients can also use LHAPI's REST API to upload programs.

#### Static content

Static content served by the LHAPI service includes the following:

* A compiled, bundled HTML file that contains the LHAPI client
* Screenshots of uploaded programs
* Videos of uploaded programs
* Profile images of makers

###### How do we serve the client?
The client is served from `static_content/client.html`.  This is a single HTML file, which has all of the application JavaScript and CSS embedded right into it.  The fact that everything is embedded means that the client does not need to make HTTP requests in order to load, which means that the client can be downloaded and ran from your hard disk, or from within another runtime such as JavaScriptCore or ChakraCore.

###### How does the static client file get there?
When you change the source code, you should have a `gulp` build process running, which is watching the files.  When the source files change, gulp will transpile the file using `babel`, and the bundle them using `browserify`, and then inject the resulting JavaScript string into a <script> tag in the client HTML.

###### How do we serve videos and screenshots?
As part of LHAPI's mission, it will automatically generate screenshots and videos of every program that a maker uploads.  It places screenshots in `static_content/program_screenshot` and videos in `static_content/program_video`.  These screenshots and videos are displayed in the LHAPI client, and also in social media tiles.

#### Dynamic content

* `/maker/[maker_id]`, to CRUD maker records: a composition of a couple database tables.
* `/affordance/[something]`, to CRUD valid JWT tokens: potentially could be zero database interaction needed with JWT.
* `/program/[program_id]`, to CRUD program records: a composition of **many** database tables
* `/[program_id]`, aliases to `/program/[program_id]`, so that programs are viewable straight from the root URL



### Database Schema

#### `maker`
| id | subtype_ref | name | bio | date_born | email | signature |
The JWT token stored on the client is a subset of these columns/properties.
The Maker entity inside the LHAPI client/server source code is a superset of these columns/properties.

#### `program`
| id | subtype_ref | maker_ref | date_born | title | summary | schema_ref | schema_data | stack_data | signature_data |

#### `progam_subtype_1` (heightmap schema)
| length_x | length_y | camera_position | camera_offset | light_position | light_offset | light_color | light_intensity |







## LHAPI client
To be completed.

### Architecture
#### Route handling
###### What is the purpose of URL routes in relation to the LHAPI single-page client?
Even though the client is rendered and navigated entirely within a single HTML page, we want to provide routes, so that existing browser bookmark functionality can be used to fill state within the app once it loads.  We can use a `#` in the URL to "fake" a URL page change, when in reality, it is just a mechanism used to fill state in the application, on the existing page.
#### UI Rendering
#### State management
#### Build stack







## Shared code, data model, and general technical concepts
The main concept to grasp within LHAPI is its actor/skill system, which uses factory functions to create plain JS objects, runs them through "skills", which attach methods/properties on them, and then serializes them to communicate with the LHAPI back-end.

Secondary concepts would be our use of straight ES5 and ES6/Babel/Browserify, to target the Node runtime and the browser runtime.



### From the database, to the wire, to the business logic, to your screen

###### From the database...
All data in LHAPI starts off as a row in a database table.  That row references other rows, which are combined together to form an `actor_record`.  The actor record is a plain JS object that contains only data that is stored, somewhere, in the database.

###### To the wire...
All actor records are just plain JavaScript objects, that contain either primitive values, or other plain JS objects that contain primitive values.  They hold data only.  All of their properties correspond 1-to-1 with columns in different tables of the database.

actor records can be easily serialized to text, compressed, and sent over HTTP, back and forth from client to server.

###### To the business logic...
When they hit the client, actor records have an actor factory function ran on them, which turns them into full-blown Actor objects.  These objects re-inflate the raw data from the actor record (beyond what JSON.parse does), and attach all sorts of nice methods and properties (called "skills").

Actor objects can be used in both the client and the LHAPI back-end.

###### To the screen!

Some Actors need to be displayed on-screen, which requires them to have even more special methods and properties.  In addition, actors are usually displayed in relation to other actors, so we need to deal with pre-fetching to limit network requests.

These needs are served by the UIActor factory, which will attach display-related properties, and also run the Actor factory functions on all pre-fetched actors they are related to.



### Actors

"Actors" in LHAPI are the same as "entities" in many other system.  Actors can have properties (data stored in their base table), fields (data stored in other tables), and skills (methods and properties that are attached when the actor is constructed).

Right now, LHAPI really only has two types of actors: `maker`, which is a user, and `program`, which is an uploaded program from an LHVM authoring program.



### Actor skills

These are special methods and properties that can be attached to the actors.  For example:
* An actor may be able to turn itself into a tree node, given a list of its possible descendants.
* An actor may be able to clone itself.
* An actor may be able to serialize itself.



### Actors and subtypes

Some actors have different configurations, which behave the same as the base actor, but have additional fields, methods, and properties attached to them.  We accomplish this using a `subtype`, which is an integer index.  All actors have the fields and properties of subtype 0, which is the base type.  Subtyped-actors (those actors with a subtype 1 or above) will get the subtype-specific fields and properties, *in addition* to the base fields and properties.


###### Why the mysterious array indexes on factory functions and actor skills?
Within the internals of LHAPI, we use the `subtype` property as an array index, when mapping entity/subtype pairs to their factory functions or actor skills.



### JavaScript utilties

The main one here is Object.filter, which will take an object as its first argument, and return a new object as its output.  The returned object only has the methods/properties of the original object, where the property name was present in any of the objects given as additional arguments.



### UIActor vs Actor vs actor_record

#### actor_record
An actor starts its life within the database of LHAPI.  It starts in its base table, where you get an incomplete actor record.  Then, its subtype is inspected, and the subtype data is attached.  Then, field data is attached.  Finally, any of the properties/fields that reference other entities are pre-fetched, and attached to read-only properties.

This data structure is called an "actor_record", and it is serialized and sent to the client.

#### Actor
When the client gets the actor record, it inspects its `actor_type` and `subtype` properties, and looks up the appropriate factory function.  The factory function will create a new object with the record's properties, and also do a few other things:
* "enhanced properties", such as date strings, will be re-inflated into moment() objects
* "struct properties", such as special data structures, will have their factory function ran on them
* all of the actor's methods will be attached to it

The actor object can then be used within business logic, inside of the client, or serialized and used to communicate with the LHAPI REST API.

Note: actor relation properties do NOT have their Actor constructor ran on them, when you run the Actor factory, because that would cause an endless cycle of factory function calls.  See UIActor below.

#### UIActor

###### Why do we need UIActor, instead of just using Actor?

*Reason 1: Prevent circular factory function calls*
Many times in the UI, you want to display actors that are related to a given actor.  For example, if you are looking at a program, you probably want to see its maker.  Normally, the client would have to make two REST API requests (one for the program, and one for the maker), but LHAPI will pre-fetch the `maker` property when you request the program.

The problem is, this is just raw data (the actor_record for the maker), and doesn't have any of the Actor properties attached to it.  So, inside the Actor factory function, we could also call the Actor factories for all of these relations, right?  We could.  The problem is, this would result in an endless cycle of Actor factory function calls.

Therefore, we have a special factory function, UIActor, that you can use at the top-level, in your UI, instead of Actor.  UIActor will call the Actor factory function on all of its related actors, but only one layer deep.

*Reason 2: Attach display-specific abilities*
The LHAPI back-end REST API runs on Node, which means that it's written in ES5.  The shared code (such as Actor factory functions) is also written in straight ES5--this makes it easier to use the code in the LHAPI back-end, and makes it easier to test, using simple tools such as `tape`.

The front-end, however, is written in ES6 with JSX, and is being compiled to ES5 using Babel.

Therefore, if you are using an Actor factory in the ES5 back-end, what happens if that Actor references a React UI component written in ES6?

Thankfully, this situation never happens, because all of the display-specific, React-specific, ES6-specific UI components are imported by UIEntity (ES6 side), not Actor (ES5 side).  The back-end can happily work with its ES5 Actor objects, blissfully unaware of the ES6 React components that are attached to various properties of the UIActor counterpart.

# T  H  E     E  N  D
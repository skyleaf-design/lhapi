# LHAPI entity/ability system
Thoughts on a possible architecture: what would LHAPI look like if it was architected with entity/ability, instead of the class/protocol paradigm that TypeScript imposes?

## Shared code, data model, and general technical concepts
The main concept to grasp within LHAPI is its entity/ability system, which uses factory functions to create plain JS objects, runs them through "abilities", which attach methods/properties on them, and then serializes them to communicate with the LHAPI back-end.

Secondary concepts would be our use of straight ES5 and ES6/Babel/Browserify, to target the Node runtime and the browser runtime.



### From the database, to the wire, to the business logic, to your screen

###### From the database...
All data in LHAPI starts off as a row in a database table.  That row references other rows, which are combined together to form an `entity_record`.  The entity record is a plain JS object that contains only data that is stored, somewhere, in the database.

###### To the wire...
All entity records are just plain JavaScript objects, that contain either primitive values, or other plain JS objects that contain primitive values.  They hold data only.  All of their properties correspond 1-to-1 with columns in different tables of the database.

entity records can be easily serialized to text, compressed, and sent over HTTP, back and forth from client to server.

###### To the business logic...
When they hit the client, entity records have an entity factory function ran on them, which turns them into full-blown entity objects.  These objects re-inflate the raw data from the entity record (beyond what JSON.parse does), and attach all sorts of nice methods and properties (called "abilities").

entity objects can be used in both the client and the LHAPI back-end.

###### To the screen!

Some entities need to be displayed on-screen, which requires them to have even more special methods and properties.  In addition, entities are usually displayed in relation to other entities, so we need to deal with pre-fetching to limit network requests.

These needs are served by the UIentity factory, which will attach display-related properties, and also run the entity factory functions on all pre-fetched entities they are related to.



### entities

"entities" in LHAPI are the same as "entities" in many other system.  entities can have properties (data stored in their base table), fields (data stored in other tables), and abilities (methods and properties that are attached when the entity is constructed).

Right now, LHAPI really only has two types of entities: `maker`, which is a user, and `program`, which is an uploaded program from an LHVM authoring program.



### entity abilities

These are special methods and properties that can be attached to the entities.  For example:
* An entity may be able to turn itself into a tree node, given a list of its possible descendants.
* An entity may be able to clone itself.
* An entity may be able to serialize itself.



### entities and subtypes

Some entities have different configurations, which behave the same as the base entity, but have additional fields, methods, and properties attached to them.  We accomplish this using a `subtype`, which is an integer index.  All entities have the fields and properties of subtype 0, which is the base type.  Subtyped-entities (those entities with a subtype 1 or above) will get the subtype-specific fields and properties, *in addition* to the base fields and properties.


###### Why the mysterious array indexes on factory functions and entity abilities?
Within the internals of LHAPI, we use the `subtype` property as an array index, when mapping entity/subtype pairs to their factory functions or entity abilities.



### JavaScript utilties

The main one here is Object.filter, which will take an object as its first argument, and return a new object as its output.  The returned object only has the methods/properties of the original object, where the property name was present in any of the objects given as additional arguments.



### UIentity vs entity vs entity_record

#### entity_record
An entity starts its life within the database of LHAPI.  It starts in its base table, where you get an incomplete entity record.  Then, its subtype is inspected, and the subtype data is attached.  Then, field data is attached.  Finally, any of the properties/fields that reference other entities are pre-fetched, and attached to read-only properties.

This data structure is called an "entity_record", and it is serialized and sent to the client.

#### entity
When the client gets the entity record, it inspects its `entity_type` and `subtype` properties, and looks up the appropriate factory function.  The factory function will create a new object with the record's properties, and also do a few other things:
* "enhanced properties", such as date strings, will be re-inflated into moment() objects
* "struct properties", such as special data structures, will have their factory function ran on them
* all of the entity's methods will be attached to it

The entity object can then be used within business logic, inside of the client, or serialized and used to communicate with the LHAPI REST API.

Note: entity relation properties do NOT have their entity constructor ran on them, when you run the entity factory, because that would cause an endless cycle of factory function calls.  See UIentity below.

#### UIEntity

###### Why do we need UIentity, instead of just using entity?

*Reason 1: Prevent circular factory function calls*
Many times in the UI, you want to display entities that are related to a given entity.  For example, if you are looking at a program, you probably want to see its maker.  Normally, the client would have to make two REST API requests (one for the program, and one for the maker), but LHAPI will pre-fetch the `maker` property when you request the program.

The problem is, this is just raw data (the entity_record for the maker), and doesn't have any of the entity properties attached to it.  So, inside the entity factory function, we could also call the entity fentityies for all of these relations, right?  We could.  The problem is, this would result in an endless cycle of entity factory function calls.

Therefore, we have a special factory function, UIentity, that you can use at the top-level, in your UI, instead of entity.  UIentity will call the entity factory function on all of its related entities, but only one layer deep.

*Reason 2: Attach display-specific abilities*
The LHAPI back-end REST API runs on Node, which means that it's written in ES5.  The shared code (such as entity factory functions) is also written in straight ES5--this makes it easier to use the code in the LHAPI back-end, and makes it easier to test, using simple tools such as `tape`.

The front-end, however, is written in ES6 with JSX, and is being compiled to ES5 using Babel.

Therefore, if you are using an entity factory in the ES5 back-end, what happens if that entity references a React UI component written in ES6?

Thankfully, this situation never happens, because all of the display-specific, React-specific, ES6-specific UI components are imported by UIEntity (ES6 side), not entity (ES5 side).  The back-end can happily work with its ES5 entity objects, blissfully unaware of the ES6 React components that are attached to various properties of the UIentity counterpart.
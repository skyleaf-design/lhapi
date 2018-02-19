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







## LHAPI business logic and shared code.

LHAPI entities use a class/protocol system provided by TypeScript--this provides static type checking, and code that is more self-documenting than the alternative entity/ability architecture.  The disadvantage is that static class/protocol systems are more verbose than dynamic object compositions (which the alternative "entity/ability system" is one of).

### Classes and protocols: client

### Classes and protocols: server

### Classes and protocols: shared

### Serialization

Done via Protobuf objects, which are constructed using factory functions that are auto-generated by `protoc`, which is ran against the protobuf descriptor files in the `lhvm-spec` repo (doesn't exist yet).

#### descriptors

### Data to back-end record process

### Data to front-end entity process









## LHAPI client
To be completed.

### Architecture


#### Route handling

For the LHAPI client, we have the concept of front-end routes, that a human would navigate to, and a back-end route, were the client can interact with REST resouces via HTTP requests.

###### front-end routes

Shows the 
`my.domain.tld/2342413253253452435`
`my.domain.tld/2342413253253452435?maker=true`

###### back-end routes

`my.domain.tld/stack/` -> REST resource for stack entities
`my.domain.tld/maker/` -> REST resource for maker entities
`my.domain.tld/affordance/` -> REST resource for JWT


###### What is the purpose of URL routes in relation to the LHAPI single-page client?
Even though the client is rendered and navigated entirely within a single HTML page, we want to provide routes, so that existing browser bookmark functionality can be used to fill state within the app once it loads.  We can use a `#` in the URL to "fake" a URL page change, when in reality, it is just a mechanism used to fill state in the application, on the existing page.


#### UI Rendering

The LHAPI UI consists entirely of the stack viewing client: there are **no** separate pages for user profiles, statistics, etc.  Everything is contained in the one page, with different sidebars, dialogs, and modals accomodating everything that is not primary content.

For the most part, the React components that comprise the LHAPI client are stateless functions.  The main exception to this are:
* Forms: these have to contain their own temporary state, to validate, before they allow the user to submit.
* WebGL visualizer: the visualizer needs a DOM reference to the canvas element, in order to start and stop the running animation, and therefore it must be stateful.

#### Main UI components

* The stack visualizer
* The maker profile
* The stack action panel (fork, edit, delete)
* The stack outline editor: right now, limited to simple, global transforms, such as slowing down time

#### State management

State is managed in Redux, even though there is no reason to pull in this library--I'm interested to see how this plays out with TypeScript.  Checking the Redux action creators and reducers will be difficult with TypeScript, but I believe the solution lies in TypeScript's **descriminated union types**.

###### Reference
[https://spin.atomicobject.com/2017/07/24/redux-action-pattern-typescript/]


#### Build stack

TypeScript -> Browserify orchestrated within a gulp build pipeline.  Notice that Babel is missing from the pipeline: we don't need it for anything, because TypeScript is a huge, monolithic compiler.





## The LHAPI build process

As part of introducing TypeScript, we have a problem: the client code targets browser ES6, the server code targets Node (most of ES6) in CommonJS format, and other code needs to be shared between the two runtimes.  In additions, we need to **debug** and **test** the code in both runtimes.

These problems are mitigated by the TypeScript compiler, module bundlers, sourcemaps, and other trickery, but it's good to remember that we are targeting two separate runtimes with a foreign language.

### How is our build process set up?

We have two separate build processes: a simple build process for the Node server-side code, and a complex process for the browser-bundled client-side code.

#### npm scripts are the orchestrator
npm commands start the build processes for the client and server.  The client build process is started by invoking gulp.  It's unclear right now how the server build process will be started: will this be tsc, gulp, nodemon, or ts-node?

#### server build process
The server code is compiled, by TypeScript, to target Node.  Sourcemaps are also output, to aid in debugging.

Somehow, while the build/dev environment is running, we need to re-start the server, every time the TypeScript is successfully compiled into JS.

#### client build process
The client code is compiled by TypeScript into ES5 modules, and then handed off to an ES5 bundler, such as WebPack or Browserify.  Sourcemaps are output, to aid debugging.  Since we are running the ES5 modules through a bundler, we need a build pipeline, handled by gulp.

#### build pipeline
gulp is used to copy static assets, start a web server to serve the client code, run the watchify/browserify transforms, etc.

We could also possibly use gulp to watch our server compilation output file, and re-start the server process when the file changes.

###### reference

[https://github.com/gilamran/fullstack-typescript]









# T  H  E     E  N  D
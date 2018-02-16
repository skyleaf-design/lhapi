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









# T  H  E     E  N  D
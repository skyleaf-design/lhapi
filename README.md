# lhapi
Internet publishing platform for LHVM-based interactive programs

## Mile-high overview

###### What is the end goal of LHAPI?
The goal of LHAPI is to provide a low-friction pathway between the creation of an LHVM program, and its consumption by other people.

###### In general, how does this work?
Makers create *LHVM programs* using an IDE such as __LiquidHex__.  They upload these programs to the **LHAPI service**, which publishes them on a **web page** for the world to view.  When people view these pages, the *LHAPI client* loads in their browser, and runs the maker's program.

###### What are the technical components of LHAPI?
LHAPI has two parts: a Node web service that runs on a server, and an HTML/CSS/JS client that runs on users' local computers.  The static client is served from the LHAPI web service, but they are separate entities.

###### What does the LHAPI service do?
The LHAPI service handles the process of authentication, program upload, program execution, and social sharing:
1. A maker creates an authenticated account with the LHAPI service.
2. A maker uploads their program, and the LHAPI service runs it in a headless browser, takes screenshots and video, and saves them to static files.
3. SEO metadata is generated, pointing to these static files, which will make your program appear prominently in social web services, if you decide to share it.
4. When people view the maker's program, the __LHAPI__ service delivers the __LHAPI__ client, which is a static HTML/CSS/JS application that loads in their web browser.
5. The LHAPI client requests the LHVM program at the linked URL in the address bar, processes it using **lhvm-js**, and then displays the result on the screen.

###### Which LHVM schemas are supported?
Only the heightmap schema.  In the future, makers will be able to choose from different LHVM stack schemas, which will run within the LHAPI client.

###### Why would I want to hack on this?
* Custom WebGL visualizers
* Add scripting hooks and workflow to your LHVM programs
* Make your LHVM programs sampleable over the web
* Add additional technical capabilities, such as socket streams
* Integrate with other web services
* Add support for other LHVM stack schemas



## Intended usage

* Use the official service hosted by Skyleaf Design
* Run and modify your own version, on your own server
* Share your modified source code with the general public

Note that the LHAPI build process will automatically bundle up the source code into an archive file, which you can serve from the LHAPI service, upload to a separate DropBox or similar service.  You can also ignore this archive, and instead, share your source code directly on GitHub or similar publicly-accessable service.

## Non-intended usage

* Not for use within another program or product, unless that program also conforms to the AGPL

If you need to use LHAPI within another program or product, contact Skyleaf Design to receive LHAPI under a more permissive license, which we will be happy to grant.

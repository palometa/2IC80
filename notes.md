npm/cli has a lot of dependencies

important ones: npm/libnpm <----- maybe stores authentication info??

which contains code for interfacing with the registry

however, the code packing and publishing a package lives in

npm/libnpmpublish

it passes opts to fetch, which include login tokens etc.


npm/npm-registry-fetch does API calls to the registry in many packages

if (2FA enabled for publishes) {
    wait for publish to attach malware
} else {
    find packages by user
    poison them all!!!!
    upload them lmao
}

interesting auth data in lib/config/get-credentials-by-uri.js (npm/cli)

how to find npm/cli data: look at bin/npm. this calls "node $basedir/node_modules/npm/bin/npm-cli.js"
on line 47: if (npm.deref(npm.argv[0])) npm.command = npm.argv.shift()
what happens to npm.command?
did everything by scouring npm/cli source code, next time should look into hardcore debugging to find out what state we can fuck up
at the bottom we call npm.load, no reference to npm.command here, but it is available in the object itself, aha
what is npm then? well, it's in /lib/npm.js
npm.load calls the load function.
working both with C++ and JavaScript in one quartile made me appreciate strong typing. WHAT ARE ALL THESE VARS  AAA
i actually did way more fruitless research than this, diving into publish and auth methods exposed in npm/libnpm npm/libnpmpublish npm/npm-registry-fetch

new approach though: git clone the npm/cli source and add breakpoints using Visual Studio Code. we can give arguments in launch.json.
we make a debug target for "program": "${workspaceFolder}\\bin\\npm-cli.js". running it with no arguments prints the help message, success!
Running it with the "-v" argument prints the version number, can we spot this using a breakpoint?
breakpoints didn't work, but this was due to a weird wrapper at the top of the file.
We can break on the first line of code and simply step through to see where commands get handled + run arbitrary expressions.
In essence we are trying to inject arbitrary lines of code ourselves, so being able to debug this is actually extremely helpful
A call to --version is actually handled in npm-cli.js as well.

to test publishing we need to change the current working directory in launch.json, cwd
with console: integratedTerminal we can do stdin, execute commands... it's perfect :D

throw breakpoint into publish.js, check the callstack to see where it was used
npm.js line 124: var cmd = require(path.join(__dirname, a + '.js'))
commands are loaded dynamically, that's why we couldn't find any mention of them!
npm-cli.js line 131: npm.commands[npm.command](npm.argv, function (err) ...
AHA so there it is

we skip node_modules as the debugger gets off track there: "${workspaceFolder}/node_modules/**/*.js", in skipFiles

when injecting code make sure the same senctence doesn't appear twice, we can check this with printing
to inject to toolkit we can use require('...'), this ensures it is loaded only once in the case that we accidently poisoned twice

now, we found the publish.js, and we want to require the toolkit, which puts itself into the package ONLY if the user is authorized, so right before gzipping. for this we need to traverse into node_modules, we found out the actual publishing is done somewhere else.
to find out where the unauthorized error is thrown we search the string in the library and look at the call stack

a search through all files for "401" actually gives a hit in CHANGELOG.md:
* [`857c2138d`](https://github.com/npm/npm/commit/857c2138dae768ea9798782baa916b1840ab13e8)
  [#20032](https://github.com/npm/npm/pull/20032)
  Fix bug where unauthenticated errors would get reported as both 404s and
  401s, i.e. `npm ERR!  404 Registry returned 401`.  In these cases the error
  message will now be much more informative.
  ([@iarna](https://github.com/iarna))

following https://github.com/npm/npm/pull/20032/files shows us which code throws the error: lib/utils/error-message.js

An error code is printed to stderr: E401. There is a case statement on which we set a breakpoint... then go down the callstack oh boy is that big, theres all this stuff from node_modules.
The error seems to come from errorHandler.apply(this, arguments). this is the npm object, and arguments somehow already contains the error message.
unfortunatly the visual studio code debugger does not support breaking on variable change, so we have to do some more digging

or... NEW APPROACH. pilfer credentials, if they are there they are probably valid and we can just assume the call to publish is gonna work.

ok let's just scrap credentials completely, there are too many strategies for auth and not enough documentation, we will never get through all of the corner cases.

deno is an alternative to node, it does proper sandboxing of file and network. node had the opportunity to do this via V8, which is an excellent sandbox due to browser security concerns. so the insecurity and need for trust in npm is inherit to node



multiple stages:
download malicious package
package executes toolkit -> source plant into npm
there are multiple ways for a package to execute it: the simplest: preinstall hook
however this is trivial to spot
we can hide it in a weird folder or maybe inject it in the main file or something

then: npm publish called -> toolkit gets attached to package... the cycle continues.

so: planting (lifecycle hooks, hidden in file) -> uploading (npm publish, proactive?)
                               ^out of scope                              ^out of scope

nice, we need to do two things: when pack is called AND on a line where the package directory is available we call malicuos pack.js
module.exports.prepareDirectory = prepareDirectory throw that in there

for report:
documenting as you go is incredibly important.
i became very good at debugging
in software dev you want to move slow and make the best thing possible but here you move fast, you'll end up writing code you aren't gonna use
this is also my first exposure to open source, it is fun to understand the tools you are using, i appreciate that people do this for free

section on next steps: more obfuscation, automatic publishing, etc. maybe i'll start hacking away at deno to see if it really is any safer
catch a LOT more corner cases: different ways of calling pack or publish, different npm versions... how will you handle updates?

`````` VIRTUAL MACHINE ``````
Using puppy linux, it is loaded in ram so we just need the snapshot? hm.
puppy linux only has a root account. this is not needed for the attack to work
Node.js is not sandboxed and gives full access to the computer's resources, and
I've tested it on Windows, it works. (but maybe say you want to edxplore this further)
``` SIKEEEEE ```


interesting problem i had: npm uses asynchronous code with bluebird promises.
However, i wrote my rewrite func to be synchronous. it didn't immideatly error but
wrote files to the wrong streams: solution: inject different promise
traditional code uses callbacks, but errors are non-standard
promises make it explicit when an error is thrown
modern javascript uses async/await, which is syntactic sugar for promises
however this is very hard to inject into the npm codebase since none of the functions there use it

alternate approach: rewrite rewrite.js to rely on synchronous fs calls

problem: on the vm node was installed by root, so we don't have permission to overwrite stuff. BUT there's a loophole: if node was installed with nvm the user has permission to write to it :)

after malware install we suddenly need root priviliges to run npm, this has something to do with the symlink. I'm not sure how to solve this now, we can demonstrate the malware using root for now.

windows vm: https://developer.microsoft.com/en-us/microsoft-edge/tools/vms/


> proc('readlink -f /usr/bin/npm')
returns /usr/lib/node_modu;es/npm/bin/npm-cli.js for some reason

this is right. malware installs succesfully but it makes the symlink in 

OK. installation on unix breaks the symlinks somehow. they still show up but are
unusable. so, npm must now be called with `node /usr/bin/npm`

IT WORKS!!!
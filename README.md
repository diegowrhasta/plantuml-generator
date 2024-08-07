# plantuml-generator

* This project aims at making use of already working solutions in order to get the 
intended result

## Use Case

The idea is to somehow have a github repository up and running, to which one can 
commit files that hold inside of it text that represents a __plant uml diagram__, 
through a Github Action, it is expected to have all that text render an svg, and 
then have that svg comitted to the same repository right after.

## Getting the source files and sending them for conversion

First things first, there are services that help when turning a diagram represented 
in syntax into working images. Services such as _Kroki_ and _PlantUML_. They have 
endpoints open to the public to which you can send your planuml syntax and then they 
turn around and respond with a working image. _(For free)_.

We can definitely host a Docker image of these services if we are concerned about 
privacy, that will definitely incur in hosting costs though.

With this premise, this solution makes use of HTTP requests to _Kroki_ specifically, 
in order to feed `plantuml` syntax and then saving images that come back from the 
server in the repo itself.

The way to consume an endpoint to then retrieve an svg back is as follows:

````
POST https://kroki.io/plantuml/svg
{
  "diagram_source": "@startuml\nBob -> Alice : hello\n@enduml"
}
````

You have to make a POST request to the endpoint for `plantuml/svg`. And in the 
request body under a `diagram_source` property we can send all the syntax. Beware 
that this will more than likely contain the full diagram description with all special 
characters, so in a non-human readable way.

## Github Actions

The start of the solution are the automation features that GitHub provides **Actions**. 
Under a `.github/workflows` folder in the repository, we can add different workflow 
descriptions for different workflows we want. In this case there's a `generate-uml-diagrams` 
that is taking care of doing the whole solution.

The solution assumes that there's some place in the repository that will hold different 
diagrams under different `.plantuml` files (the extension won't matter, but for consistency's 
sake we will state that we should have only files under this extension with contents that 
represent a renderable diagram for the server).

After the contents of all the files are read we then make nth-post requests to 
_Kroki_ and per each response we save that `svg` file under a `svgs` folder at the 
root of the repository. By the end, the action takes care of comitting all these 
generated svg files.

## Configurable Action

The action will only run if one triggers it manually, or if the commit message at 
the HEAD has somewhere in between `[RUN DOCS]`. This will be a flag in the commit 
message that we can put so that we run this workflow, since it is costly, we are 
making requests to a server, we are then saving those responses in the repo. So we 
should be running this workflow only when we actually need to re-generate docs.

## Breaking down the action

We will break down everything the action does so that it's extremely clear what 
each sentence is doing and why it is doing it that way.

### Initial Conditions

This workflow will be initially triggered with pushes that happen on the **main** 
branch, and pull requests that have the **main** branch as the **base**.

### Circuit Breaker

As mentioned before we have `[RUN DOCS]` flag requirement on the use case, and so 
the workflow does an initial check on the `process-docs` job that's the one that 
does the whole bulk of the logic. If we do not have this string somewhere in the 
commit message the job will not run at all.

### Steps

#### Checkout code

We make use of an already baked in Github Action `actions/checkout@v4` that makes 
sure we clone and checkout the repository we will start running the logic in. We 
also want for the github credentials to persist so that we then later do commits and 
stuffs, so we use the `persist-credentials: true` parameter.

#### Read and process files

This is the most important step of the job, it will read the files with syntax, and 
then use those same contents to make calls to the plantuml server and then save the 
responses in files that will then be comitted to the repository.

- Initially we add an id to the task since this can help reference later down the line 
in other tasks the same job and the variables that might have been created in it.
- We create an `svgs` folder in case there isn't one, and in case there's we just 
let it be.
- We then declare a structure that will attempt to read all files that are inside 
of the `docs` folder.
- We make a check to see if the iteration entry is an actual file (since the loop 
can also retrieve directories). Only then we will process the file.
- We then extract to pieces of information the filename without path (`basename`) 
and then the contents of said file (`cat`) these results are saved under two variables. 
- We then save under a variable the name that will take the svg file, this uses syntax 
so that we remove the extension and only are kept with the file name and we then append 
to it `svg`
- We then make use of the special `$GITHUB_ENV` file that is used to save under different 
variables text that holds special characters in its contents. Firstly we save the 
filename. And then in order to properly save the content with multi-line content (that's 
possible) we break down the insertion of the content in three statements one that 
firstly saves the variabel name and then appends an `EOF`, we then paste the multi-line 
content raw, and lastly we append another `EOF` to mark the beginning and end of the 
value of this _file\_content_ variable.
- We then get ready to make a POST request with CURL, we simply pass down the file content value 
that we have to a `diagram_source` variable.
- We then assign to another variable the string that will be the payload, we use 
`jq` which is a utility to make json objects, it starts at null, we then add to the 
command's runtime a new variable `ds` that will have all the content from our plantuml 
syntax, and lastly we are building the json object with a `diagram_source` key and 
then as a value all the content under the previously assigned `ds` variable.
- We then do a POST with curl to the specified endpoint, and send as a header 
`Content-Type: application/json` so that the server understands we are sending 
json as payload. And we then feed onto the payload our variable we calculated before, 
and we also state that the response should be saved under the name of the previously 
calculated `svg_file` string.
- Lastly we will take that generated file that's at the root right now and we will 
pass it onto the staging section of the repository

### Commit SVG files

This step is more straightforward, there's a `common` user.name and user.email 
for a github action bot when comitting files to a repository, we are assigning such 
names to the respective global variables so that we can later see these commits under 
the bot's name and without git throwing an error.

We make a check in here before trying to commit and push so that the workflow is 
idempotent, meaning that it should be successful at the first run and all subsequent 
runs. In case this workflow is run, and no new diagram was actually generated, 
git won't pick up on differences and it will fail, this makes it so that it just 
finishes gracefully. In case differences are picked up on, it will then attempt to 
commit all the files that are in staging, which should be the `svg` generated images.

One important part in here is the `GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` section, 
that assigns to the env variable under `GITHUB_TOKEN` the value of the token that was 
fed to the workflow under `secrets.GITHUB_TOKEN`, this is just to make sure we have 
the token correctly assigned, this token actually holds the permissions the Actions 
bot has on the repo at that current point in time.

### About Actions comitting to the repo

Beware that by default an Action **_cannot_** write to the repo unless you give it 
specific permissions to do so. This is configured under Settings > Actions > Read and write permissions.

### Extra Notes

- One point that was sanitized was that Github is going to deprecate a `::set-output` 
command that is used when trying to save contents with special characters to env vriables. 
And so this action makes use of the new `$GITHUB_STATE` env variable syntax.
- Another point that also popped up is that the _checkout_ action that we use at the 
beginning was using a deprecated version of node. So, when switching to the `v4` of 
said action said warning went away.

## Sources

- [Asciidoc with Plantuml](https://dev.to/anoff/preview-asciidoc-with-embedded-plantuml-in-vs-code-3e5j)
- [Asciidoc Diagram extensions](https://docs.asciidoctor.org/browser-extension/diagrams-extension-quickstart/)
- [Kroki install](https://kroki.io/#install)
- [Kroki usage](https://docs.kroki.io/kroki/setup/usage/)
- [Kroki HTTP clients](https://docs.kroki.io/kroki/setup/http-clients/)
- [Kroki use cases](https://kroki.io/examples.html#use-case)
- [Plantuml and PDF](https://fiveandahalfstars.ninja/blog/2017/2017-05-01-plantuml-and-pdf)
- [Github deprecates set-output](https://github.blog/changelog/2022-10-11-github-actions-deprecating-save-state-and-set-output-commands/)
- [Github's node support on actions](https://github.blog/changelog/2024-03-07-github-actions-all-actions-will-run-on-node20-instead-of-node16-by-default/)



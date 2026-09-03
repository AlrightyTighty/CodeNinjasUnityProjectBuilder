# CodeNinjasUnityProjectBuilder
Tool that builds unity projects to a visible s3 link so parents are able to see their kids' progress.

## User Stories

- As a user, I want to create new projects through this app, replacing the need for the unity hub.
- As a user, I want to view projects I have created through this app and open them.
- As a user, I want to upload changes from projects I have created through this app.
- As a user, I want to choose to create a new project from an already existing unity project.
- As a user, I want to download projects I have created through this app.
- As a user, I want to only view, upload, and download projects that I myself own.
- As a user, I want to download specific project versions.
- As a user, I want to choose to build a given project version to a playable link.
- As a user, while I am waiting for my project to be built, I want to see its queue status.
- As a user, I want to delete past project builds.


## Project Architecture


### API and Lambdas

The entire API will be run on one lambdalith (/lambda). This lambda will be written in typescript and tested with vitest. Fastify is used to wrap the entire API into a lambda using the official `@fastify/aws-lambda` package. The API will be called directly with its lambda function URL.

Other lambdas will be written as short independent handlers with a few tests. We will deploy code via github actions.

### Build Server

We will use a github push webhook on the ec2 server to update it. Each time the server wakes up, it will git pull from this codebase and read from the `server` folder and run the docker image. This will also be written in typescript and tested with vitest.

### Project/Build Storage

Projects will be zipped instead of being saved as a full directory. There is no need to granularly access each file when on s3, and it makes it easier for parents to install the projects at home if they come pre-zipped.

Builds will have the full build output within one prefix. We will use CloudFront in front of the build folders to allow for quicker and smoother access and analytics monitoring.

Project and build storage will be handled on s3. The object prefix will be `projects/ninjalogin/projectname`. For this reason, project name is unique per ninja. An example key for a project would be, `projects/john.doe/my-project/project.zip`. Builds are version-dependent, and building a new version shouldn't invalidate old versions. For this reason, build prefixes will include the version. The build prefix will be `builds/ninjalogin/projectname/version`.

### Authentication
I have provided an api at POST api.codeninjaswestkaty.com/auth/token that you can use for this project. This API endpoint will give you a JWT (if you don't know what this is, go to [jwt.io](https://www.jwt.io) and mess around!)

Your API will be responsible for verifying the JWT is signed properly. You can get the public key at GET api.codeninjaswestkaty.com/auth/publickey

### Building
Upon requesting a build, the API lambda will insert a new message into a build queue on SQS and wake up the build server. The build server is based on a spot-priced c6i instance that we wake up ad-hoc. The build server will process builds from the queue until it is empty, in which case it will shut itself down. Once a build is complete, the build will be pushed to an s3 bucket as described above.

#### Edge Case: shutdown as build is added
Since it is possible to race condition the shutdown, we will have a cron job that triggers a lambda that checks if the build queue isn't empty. If it isn't, it will send the wakeup message to the build server. This cron job will run every 10 minutes. 

Lambda free tier has 400,000 GB-seconds of execution and 1 million requests per month. A check every 10 minutes is 4380 requests per month and will likely not exceed 30000 GB-seconds, let alone 400,000, so this will not rack up execution costs.

### Versioning

S3 has built-in versioning, here is how versioning will work:

S3 versions are NOT human-readable. They are randomly-generated strings. We will need to store metadata about the version (version number, date, possible user-supplied identifier like name/change comment).

This will be done through AWS DynamoDB. Primary Key is composite on index as the partition key and version as the sort key.

When a new object is inserted into the S3 build bucket, we take the prefix and query the dynamo table for versions. We'll find the latest version, increment it by one, and insert our new version item.

S3 versioned buckets also do not have instant deletion; a deleted item is given a delete marker (soft delete), but still exists and you will be billed for it. We will want to permanently delete these objects

### Desktop App
The frontend design doesn't matter too much, go crazy. This application does have to run on windows 11, so no XCode. You guys have creative freedom here.
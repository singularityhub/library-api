# Library API

The library API is a set of registry interactions that allow for pull and push
of containers. The API includes:

 - the specific interactions between a client and registry to authenticate, push, and pull
 
The API does *not* include

 - client implementation details for how to store and manage credentials
 - registry implementation details for handling details of user accounts, authentication, or storage
 - strong requirements about fields required. E.g., many fields might be present in the response that a registry ignores.

The API strongly encourages but does not require authentication, and the spec describes the core
functionality to interact with a `library://` supported client.
Thus, while the API spec is not scoped to include the exact client logic (which
is up to the implementer) we provide examples using the open source [Singularity](https://github.com/sylabs/singularity)
client that implements the client for the developer user. To provide an understanding
of this API based on the user workflow, we provide the API endpoints needed alongside 
each step of the process to authenticate, push, and pull a container.

## API Functions

### Remote Add

#### Suggested Client Interaction

An example client login for a registry server running at `127.0.0.1` (localhost) might look like this:

```bash
$ singularity remote add --no-login DinosaurCloud 127.0.0.1
INFO:    Remote "DinosaurCloud" added.
```
 
This is a function that is completely on the side of the client, and does not involve the registry
server, so its shown for a suggested implementation only. While this is not required to implement the API, we would generally suggest that a client
have an easy way to store a named endpoint, and then list endpoints that are known.

```bash
$ singularity remote list
```

### Config Endpoint

#### GET /version

During push of a container, the client often checks a version endpoint, and the response should generally provide the API version (with prefix 1 or 2).
For the following endpoints, those that start with `/v1` indicate an earlier version of the API, and `/v2` indicate a later version. It is recommended to support version 2.0, as the implementations of the library API provide the full set of endpoints from both.

**200 Response**

```python
{
    "data": {
        "version": "v1.0.0",
        "apiVersion": "2.0.0-alpha.1"
    }
}
```

#### GET /assets/config/config.prod.json

The server `MUST` provide a metadata endpoint at `/assets/config/config.prod.json` that
provides basic metadata about the implementation. 

**200 Response**

An example response to a GET request might look like the following:

```json
{
    "env": {
        "name": "prod"
    },
    "logging": {
        "console": true
    },
    "libraryAPI": {
        "uri": "http://127.0.0.1"
    },
    "keystoreAPI": {
        "uri": "http://127.0.0.1"
    },
    "tokenAPI": {
        "uri": "http://127.0.0.1"
    },
    "auth": {
        "issuer": "http://127.0.0.1",
        "requireHttps": false,
        "redirectUri": "",
        "clientId": "services-frontend",
        "scope": "openid email profile",
        "silentRenew": false,
        "silentRenewUrl": ""
    }
}
```
Each field is explained in detail in the following table:

| Name | Type |Description |
|------|-----|------------|
| env.name  | string | A name that describes the environment (e.g., prod indicates production) |
| logging.console | boolean | Should the client provide console logging (note this does not have to be followed by the client |
| libraryAPI | string | the endpoint for the library API |
| libraryAPI | string | the endpoint for the keystore API (not required) |
| tokenAPI | string | the base endpoint for the token API (functions included in the library API) |
| auth.issuer | string | the issuer of the authentication, which can be the same or a different server |
| auth.requireHttps | boolean | Does the registry require https for interaction? |
| auth.redirectUri | string | if defined, the client should redirect here |
| auth.clientId | string | an identifier for the client, returned to the client |
| auth.scope | string |the scope of authentication used (will depend / vary by registry) |
| auth.silentRenew | bool | does the registry support silent renewal of tokens? |
| auth.silentRenewUrl | bool | if true, a URL for that (leave blank to omit, not required |

Largely, the most essential fields above are for the endpoints for different APIs that your registry supports.
In particular we are documenting the library and token API here (which seems to be a part of it).

### Login

#### Login Suggested Client Interaction

The first interaction with the Library API will be a login, or authentication with the server.
To make this possible, the implementer is suggested to provide a user interface to retrieve a token
associated with their account. E.g.,:

1. Register for the service
2. Login to the interface
3. Retrieve an API token from a profile or settings

At this point, an interaction with the Singularity client might look like:

```bash
$ singularity remote login DinosaurCloud
Generate an access token at http://127.0.0.1/auth/tokens, and paste it here.
Token entered will be hidden for security.
Access Token: 
INFO:    Access Token Verified!
INFO:    Token stored in /home/vanessa/.singularity/remote.yaml
```

And the user will copy paste the token from the interface to provide to the client.
Note that it is not required to use `http://127.0.0.1/auth/tokens` as the endpoint for providing
the user interface with a token - this is under the decision of the registry implementation (as it may
be the case that interaction is entirely command line based).
 
It's suggested that this interaction be only supported for https, however the client can optionally choose
to provide a flag to disable it, for example, for the developer use case.
It is out of scope for this specification for how the client should store the token (and indeed it doesn't need to be
required) but it does help the user with easier functionality to keep it in a private store or cache.
As an example, the Singularity software stores it in a hidden `.singularity` directory
in the user home under `remote.yaml`.

```
$ cat $HOME/.singularity/remote.yaml Active: DinosaurCloud
Remotes:
  DinosaurCloud:
    URI: 127.0.0.1
    Token: 1eb5bcrdaeca0f5a215ef242c9690209ca0b3d71
    System: false
  SylabsCloud:
    URI: cloud.sylabs.io
    Token: hahhaayeahrightdude
    System: true
```

#### Login API Endpoints

##### GET /v1/token-status

This endpoint provides basic authentication, where the client is expected to provide the token in a standard "Authorization" header
before continuing with further interaction. This interaction tends to preceed any registry interaction,
and can also be used by the client to validate or check a token (as shown with the login command above).
It's up to the client to drive this authentication workflow. This means setting the token in an Authorization header that might look like the following
(in Python):

```python
headers = {"Authorization": "bearer xxxxxxxxxxxxxxxxxxxxxxxxxxx"}
```

and then doing a `GET` request to the endpoint to authenticate.


**200 Response**

A 200 response from the server indicates that the token was successful to authenticate the user.


**404 Response**

An unsuccessful response should be 404, or not found. We do this generally over another 40X response
to not give the user or client any extra information about having found a partial login.


### Push

The first (and one of the primary) interactions that a client might want to do is push a container image to
a registry.

#### Push Suggested Client Interaction

Using the singularity software, a push might look like this. The Singularity software has decided to provide
an easy `remote login` command to ensure that the right credential is selected. That is again up to the client, but for
Singularity it looks like this:

```bash
$ singularity remote use DinosaurCloud
INFO:    Remote "DinosaurCloud" now in use.
```

This action is again not under the scope of this specification, but recommended for a client to allow
a user to easily interact multiple times with a registry without needing to specify it directly.
A push of the `busybox.sif` container to the namespace `<username>/<collection>/<image>:<tag>` might then look like this:

```bash
$ singularity push -U busybox_latest.sif library://vsoch/dinosaur-collection/another:latest
WARNING: Skipping container verification
780.0KiB / 780.0KiB [==============================================================================] 100 % 43.8 MiB/s 0s
```

It is up to the registry implementation to decide if the collection must exist beforehand, and some registries MAY decide to implement a separate
endpoint to create a collection programatically, however this is not part of the library API. This means that the push workflow
can take one of two forms:

##### 1. Automatic collection creation

 - Hit the [config endpoint](#config-endpoint) to derive metadata and URLs for the server
 - Authenticate the user token via the [token status](#login-api-endpoints) endpoint to authenticate the user, continue if success.
 - Look up the username via the username entity endpoint `GET /v1/entities/<username>`.
 - Look up the collection via the entity collection endpoint `GET /v1/collections/vsoch/dinosaur-collection`
 - If it doesn't exist, request to create it via `POST /v1/collections`
 - Continue to [shared push steps](#shared-push-steps)

##### 1. Manual collection creation

 - Hit the [config endpoint](#config-endpoint) to derive metadata and URLs for the server
 - Authenticate the user token via the [token status](#login-api-endpoints) endpoint to authenticate the user, continue if success.
 - Look up the username via the username entity endpoint `GET /v1/entities/<username>`.
 - Look up the collection via the entity collection endpoint `GET /v1/collections/vsoch/dinosaur-collection`
 - If it doesn't exist, stop (e.g., return 404)
 - If it does exist, continue to [shared push steps](#shared-push-steps)

##### 2. Shared push steps

 - Get metadata for the collection image (existing or not) `GET /v1/containers/<username>/<collection>/<image>`
 - Pre-push the container image `GET /v1/images/<username>/<collection>/<image>:<sha256>?arch=<arch>` to retrieve an id `<imageid>`
 - Check the version endpoint `GET /version` and act appropriately - providing a version 2.0.0x implementation is likely required.
 - Ask the registry if multipart uploads are supported: `POST /v2/imagefile/<upload_id>/_multipart` if 404, proceed to PUT blob. If 200, proceed to multipart.

After the above steps, there can be two kinds of upload: a single "PUT" of an object directly to storage, OR a multipart upload, if the registry supports it. Both are discussed in the following sections, and proceed with either of step 3 below.

#### 3. PUT blob

This flow is used if the registry does not support multi-part uploads. It's considered a deprecated endpoint because the multipart functionality is so much
better, but clients can still choose to support it.

 - Request a URL to push the image to `POST /v2/imagefile/<imageid>`, which returns a URL to send the blob 
 - Upon completion, `PUT /v2/imagefile/<imageid>/_complete` to indicate completion.
 - Look up tags for the image `GET /v1/tags/<imageid>` and update them `POST /v1/tags/<imageid>`

#### 3. Multipart upload

 - `POST to /v2/imagefile/<uploadid>/_multipart` and get 200 response with metadata needed to generate signed URLs
 - repeatedly `PUT /v2/imagefile/<uploadid>/_multipart` with upload ID to return signed urls for each part.
 - complete the upload with `PUT /v2/imagefile/<uploadid>/_complete`
 - cancel at any time with `PUT /v2/imagefile/<upload_id>/_multipart_abort`

#### Push API Endpoints

##### GET /v1/entities/username

The purpose of this view is to look up metadata associated with an entity, which is often a user account (e.g., vsoch in the examples
previously) but does not necessarily have to be. The get request should provide the query parameter for the entity name (or `<username>`)
as the primary parameter in URL.

**200 Response**

A successfully found entity should return a 200 response with the entity metadata:

```python
{
    "collections": [
        "1"
    ],
    "deleted": false,
    "createdBy": "",
    "createdAt": "2021-12-15T23:36:33.566818Z",
    "updatedBy": "",
    "updatedAt": "2021-12-15T23:36:33.609420Z",
    "deletedAt": "",
    "id": "1",
    "name": "vsoch",
    "description": "vsoch",
    "size": 0,
    "quota": 0,
    "defaultPrivate": false,
    "customData": ""
}
```

Each field is explained in detail in the following table:

| Name | Type |Description |
|------|-----|------------|
| collections | list | A list of collection identifiers |
| deleted | boolean | Whether the entity has been deleted (registries can choose to simply not return an entry for a deleted entry) |
| createdBy | string | If relevant for the namespace, an entity that created this one (not required) |
| createdAt | timestamp | the timestamp when the entity was created |
| updatedBy |  string | If relevant for the namespace, an entity that updated this one (not required) |
| deletedAt | timestamp | If deleted entities are exposed, the timestamp when the entity was deleted |
| id |  string | the identifier for the entity |
| name | string | the name of the entity |
| size | int | the size of the entity stored data (0 can indicate not relevant or not used) |
| quota | int | the size of the entity quota (0 can indicate no quota / not relevant) |
| defaultPrivate | boolean | Whether the entity collections are set to private by default |
| customData | string | custom data stored for the entity, if desired for an implementation |

Setting a quota (and keeping track of the size) is optional for a registry.

**404 Response**

The named entity was not found in the registry, and the interaction stops. This response can also be returned if:

 - no authentication token was provided
 - the associated user was not found
 - the user found was not linked with the token provided
 

##### GET /v1/collections/username/collection

This is the basic endpoint to determine if a collection exists.

**404 Response**

An indication that the collection does not exist. If the registry allows, the client can next make a `POST` request to `/v1/collections`
to create the collection.


##### GET /v1/collections

**403 Response**

The token provided was not valid.

**200 Response**

A list of [collection entities](#collection-response).


#### POST /v1/collections

This endpoint is optionally implemented by a registry that wants to support automated creation of collections.
An authorization token, as per the other endpoints, is strongly recommended.
This decision is up to the registry implementation. The body of the request should include the following parameters:

| Name | Type |Description |
|------|-----|------------|
| entity | string | The entity or user id to look up the owning entity (e.g., an id of 1 would map to vsoch in our example) |
| name | string | the new collection name |
| private | boolean | optional boolean to indicate if it should be private (should default to a registry configured default for new collections |

An example (loaded) body might look like:

```python
{
    "entity": "1",
    "name": "collection-name",
    "private": true
}
```

**403 Response**

A 403 response can be returned if the token is not valid, OR if the token does not match to the user account,
OR the collection already exists. Since these responses are expected to be sent back to a client to parse,
we provide a structured error object with a code and message.

```python
{
    "error": {
        "code": 403,
        "message": "Token not valid"
    }
}
```

If the registry wants to distinguish between the two cases, a different message about authentication being denied for the given
token can be provided. For example, here is a message to alert the user that the collection already exists:

```python
{
    "error": {
        "code": 403,
        "message": "A collection named 'collection-name' exists already!"
    }
}
```

If the client is responsible for issuing the request to create the collection after it is found to not exist, you'd
only hit this case in some kind of race condition, which hopefully isn't likely for pushing an image, which is typically a single or manual action.


**400 Response**

An "invalid payload" response can be returned if either the entity id or collection name is not provided.

```python
{
    "error": {
        "code": 400,
        "message": "Invalid payload."
    }
}
```

For all of the above, the specific message can be further customized by the registry.
Upon creation of the collection, the user associated with the token is suggested to be added as an owner of
the collection, so the user can be validated to push to it at a later time point. The private parameter
should determine the collection's level of privacy, and a collection secret can be generated if needed for 
other functionality not covered in this API.

**200 Response**

<a id="collection-response"></a>

Successful creation of a new collection should result in a 200 response with the following metadata:

```python
{
    "data": {
        "containers": [],
        "createdAt": "2021-12-16T02:15:22.446286Z",
        "createdBy": "1",
        "customData": "",
        "deleted": false,
        "deletedAt": "",
        "description": "New-collection Collection",
        "entity": "1",
        "entityName": "vsoch",
        "id": "2",
        "name": "new-collection",
        "owner": "1",
        "private": false,
        "size": 0,
        "updatedAt": "2021-12-16T02:15:22.462322Z",
        "updatedBy": "1"
    }
}
```

| Name | Type |Description |
|------|-----|------------|
| containers | list | A list of container ids associated with a collection. A newly created collection will have an empty list. |
| createdAt | timestmap | The datetime when the collection was created |
| createdBy | string | the entity (or user) identifier that created the collection |
| customData | string | custom data associated with the collection (not required) |
| deleted | boolean | If a registry chooses to expose deleted collections, this could be true |
| deletedAt | timestamp | If a registry supports exposing deleting collections, this is the timestmap of deletion. |
| description | string | A description of the collection (can have a set default, as shown above, editable by the user in the user interface |
| entity | string | the id of the entity that created the collection (can be the same as createdBy) |
| entityName | string | the string name of the associated entity (user) |
| owner | string | the identifier of the owner of the collection (often the same entity but doesn't have to be) |
| private | boolean | whether or not the collection was created as private |
| size | int | the size of the collection (typically bytes or MB) |
| updatedAt | timestmap | The datetime when the collection was updated |
| updatedBy | string | the user identifier that updated the collection |

#### GET /v1/containers/username/collection/image

This is a named collection container view, where generally the client is asking to return metadata for a specific container.
Token validation is recommended, as per the other endpoints. This request should provide the following parameters
in the url:

| Name | Type | Description |
|------|-----|--------|
| username | string | the entity or username that owns the collection |
| collection |  string |the name of the collection | 
| container | string | the name of the container image |

An authentication header, as specified previously, is also required to ensure that the user
making the request has permission to write to the collection of interest.

**403 Response**

A 403 response is returned if the token is not valid, meaning it isn't associated with any
user, or isn't provided, or is associated with a different user than provided. A 403 is also
returned if the token is valid for the user, but the user is no longer indicated to be a collection
owner.

**404 Response**

A 404 response is provided if the collection in question does not exist, as looked up by name.

**200 Response**

A successful response is an indication that the authenticated user has permission to write to the collection
and can proceed:

```python
{
    "containers": [],
    "createdAt": "2021-12-16T03:35:14.884055Z",
    "createdBy": "1",
    "customData": "",
    "deleted": false,
    "deletedAt": "0001-01-01T00:00:00Z",
    "description": "Dino-collection Collection",
    "entity": "1",
    "entityName": "vsoch",
    "id": "3",
    "name": "dino-collection",
    "owner": "1",
    "private": false,
    "size": 0,
    "updatedAt": "2021-12-16T03:35:14.899745Z",
    "updatedBy": "1",
    "archTags": {
        "amd64": {}
    },
    "collection": "3",
    "collectionName": "dino-collection",
    "downloadCount": null,
    "fullDescription": "Dino-collection Collection",
    "imageTags": {},
    "images": [],
    "readOnly": false,
    "stars": 0
}
```

The following fields are shared by a collection response:

| Name | Type |Description |
|------|-----|------------|
| containers | list | A list of container ids associated with a collection. A newly created collection will have an empty list. |
| createdAt | timestmap | The datetime when the collection was created |
| createdBy | string | The entity (or user) identifier that created the collection |
| customData | string | Custom data associated with the collection (not required) |
| deleted | boolean | If a registry chooses to expose deleted collections, this could be true |
| deletedAt | timestamp | If a registry supports exposing deleting collections, this is the timestmap of deletion. |
| description | string | A description of the collection (can have a set default, as shown above, editable by the user in the user interface |
| entity | string | The id of the entity that created the collection (can be the same as createdBy) |
| entityName | string | The string name of the associated entity (user) |
| owner | string | The identifier of the owner of the collection (often the same entity but doesn't have to be) |
| private | boolean | Whether or not the collection was created as private |
| size | int | The size of the collection (typically bytes or MB) |
| updatedAt | timestmap | The datetime when the collection was updated |
| updatedBy | string | The user identifier that updated the collection |

And the following fields are added in the case of creating a container:

| Name | Type |Description |
|------|-----|------------|
| id | string | The identifier for the container |
| archTags | dict | A dictionary of key value pairs, where the keys are the architectures supported |
| collection | The identifier of the collection |
| collectionName | The name of the collection |
| fullDescription | A full description of the collection |
| imageTags | A lookup of tags for an image (only populated if an image already exists) |
| images | A list of image identifiers associated with the collection |
| readOnly | If the image is read only and cannot be pushed to (optionally supported) |
| stars | starts for the container |

In practice, this view returns data for a collection (and containers) even if the container doesn't exist (yet).
The view itself is really a first step in authenticating to retrieve a URL to push a container to, and we
are ensuring that the user has permission to write to the collection. In practice this might mean there
are different operations that go on the back end of each registry server, but regardless of these individual
choices, the response is the same.

#### GET /v1/images/username/collection/image:sha256?arch=arch

<a id="get-image-view"></a>

This is the pushed named container view, and this is the first time the client provides the full container unique resource identifier
(`<username>/<collection>/<image>:<digest>?arch=<arch>` as url parameters.

**403 Response**

A 403 response is returned if the token is not valid, or the user affiliated with the token is found to not be a collection owner.

**404 Response**

A 404 response is provided if the collection in question does not exist, as looked up by name.

**200 Response**

The registry implementation, upon verifying that the user has permission to push to the collection, typically
will create a record of the container in the database, meaning a record of:

 - the collection
 - the container name
 - the container architecture stored somewhere (e.g., as metadata)
 - the container tag (typically a dummy tag since the client hasn't pushed any known tags yet)
 - the container digest or version (typically a sha256 digest)

And the following successful response data:

```python
{
    "data": {
        "deleted": false,
        "createdAt": "2021-12-16T03:49:22.987635Z",
        "createdBy": "1",
        "updatedBy": "vsoch",
        "owner": "1",
        "id": "5",
        "hash": "sha256.b059b0fdfe7ef1dcd561caa5e08c96c763df5992b8afb98ad32e1bac4d033787",
        "description": "D-collection Collection",
        "container": "sha256.b059b0fdfe7ef1dcd561caa5e08c96c763df5992b8afb98ad32e1bac4d033787",
        "arch": "amd64",
        "fingerprints": [],
        "customData": "",
        "size": null,
        "entity": "1",
        "entityName": "vsoch",
        "collection": "4",
        "collectionName": "d-collection",
        "containerName": "container",
        "tags": [],
        "containerStars": 0,
        "containerDownloads": 0
    }
}
```

| Name | Type |Description |
|------|-----|------------|
| deleted | boolean | If a registry chooses to expose deleted containers, this could be true |
| createdAt | timestmap | The datetime when the container was created |
| createdBy | string | The entity (or user) identifier that created the container |
| updatedBy | string | The user identifier that updated the container |
| owner | string | The identifier of the owner of the container |
| id | string | The identifier for the container |
| hash | string | the content digest (sha256) of the container |
| description | string | A description of the collection associated with the container (registries can choose to provide editing of descriptions on the level of containers too) |
| container | string | the same as the digest |
| arch | string | the container architecture |
| fingerprints | list | Fingerprints associated with the container (if applicable, optional) |
| customData | string | a string of custom data for the container (optional) |
| size | int | a container size (if known yet) |
| entity | string | The id of the entity that created the container (can be the same as createdBy) |
| entityName | string | The string name of the associated entity (user) |
| collection | The identifier of the collection |
| collectionName | The name of the collection |

| customData | string | Custom data associated with the collection (not required) |
| deletedAt | timestamp | If a registry supports exposing deleting collections, this is the timestmap of deletion. |
| description | string | A description of the collection (can have a set default, as shown above, editable by the user in the user interface |
| entity | string | The id of the entity that created the collection (can be the same as createdBy) |
| entityName | string | The string name of the associated entity (user) |
| private | boolean | Whether or not the collection was created as private |
| size | int | The size of the collection (typically bytes or MB) |
| updatedAt | timestmap | The datetime when the collection was updated |
| tags | list | tags ascribed to the container (happens after successful push) |
| containerStars | int | the number of starts associated with a container |
| containerDownloads | int | the number of downloads for a container |

This of course is a suggested implementation - and it cannot be known how different registries do this
implementation. It also is likely the case that many of these fields won't be used by a client. E.g., stars and downloads
is likely for informational purposes only, and is not directly needed by some client using the library endpoint.
For Singularity Registry Server, a temporary tag is generated that will be replaced with
the final allocated tag after upload. The format of this temporary tag can then be used to easily clean up
partially pushed (or failed) images with some kind of scheduled job, if desired. This again is not part of the
library API specification, but a note added for clarity to be helpful to developers.

#### POST /v2/imagefile/upload_id/_multipart

This is an endpoint that gives a registry power to say "Yes, I support multipart uploads" or "No, I do not."
This functionality was added to the Singularity scs client in February of 2020. In a request that is doing multipart,
the following should be provided in the body:

| Name | Description |
|------|-------------|
| filesize | the filesize (in bytes) of the upload to calculate the number and size of parts |

When the registry receives the filesize, an upload size allowed can be calculated as follows:

```
filesize = body.get("filesize")
max_size = 500 * 1024 * 1024
upload_by = int(filesize / max_size) + 1
```

At this point, given that we are proceeding with the multipart upload (a known API) we can prepare a request to the client
That includes a storage identifier (likely the container unique resource identifier and digest, it's recommended to have a consistent
function to do this for any container) and then to initiate the multipart upload to retrieve an UploadID. That might look like this:

```python
# Key is the path in storage, MUST be encoded and quoted!
storage = container.get_storage()

# Create the multipart upload
res = s3.create_multipart_upload(Bucket=MINIO_BUCKET, Key=storage)
upload_id = res["UploadId"]
```

The registry can then calculate the total number of parts needed:

```
# Calculate the total number of parts needed
total_parts = len(range(1, upload_by + 1))
```

And save these parameters with the container instance in the database. E.g.,

```
# Save parameters with container
container.metadata["upload_id"] = upload_id
container.metadata["upload_filesize"] = filesize
container.metadata["upload_max_size"] = max_size
container.metadata["upload_by"] = upload_by
container.metadata["upload_total_parts"] = total_parts
container.save()
```

Since we've interacted with the multipoint upload API to generate the upload, the only thing we need to return to the client
is the UploadID (see response 200).

**Response 403** 

Multipart is supported, but the user is not authenticated.

**Response 404**

A 404 response indicates that there is not support for multipart uploads, and a legacy upload (a single PUT) should be initiated
via `POST /v2/imagefile/<imageid>` (further below).

**Response 400**

Can be returned if filesize is missing from the body.


**Response 200**

A 200 response happens after we initiate the upload with the multipart server, and then we just
need to tell the calling client about it, specifically the uploadID to use and the number of parts and part size.

```python
# Start a multipart upload, telling Singularity how many parts and the size
{ "data": {
     "uploadID": upload_id,
     "totalParts": total_parts,
     "partSize": max_size,
  }
}
```

#### PUT /v2/imagefile/uploadid/_multipart

After a multipart upload is initiated with `POST /v2/imagefile/<upload_id>/_multipart`,
this endpoint is queried with metadata in the body, including:

| Name | Type | Description |
|------|-------------|
| uploadID | string | the uploadID known to the registry server |
| sha256sum | int | the sha256sum of the blob |
| partSize | int64 | the content size of the part, in bytes |
| partNumber | int | the part number (sent by the client) |

Upon receiving this information, the registry server should validate that the upload id
associated with the container in question is the same one stored with its metadata. E.g.,
it could be that between making the request and initiating this interaction, the upload was cancelled
and the upload id changed, although unlikely. The part number `partNumber` must also be provided
in order to generate a signed URL. That might look like this (in Python, when using Minio):

```python
# Generate pre-signed URLS for external client (lookup by part number)
signed_url = s3_external.generate_presigned_url(
    ClientMethod="upload_part",
    HttpMethod="PUT",
        Params={
            "Bucket": MINIO_BUCKET,
            "Key": storage,
            "UploadId": upload_id,
            "PartNumber": part_number,
            "ContentLength": content_size,
        },
        ExpiresIn=timedelta(minutes=MINIO_SIGNED_URL_EXPIRE_MINUTES).seconds,
)
```

And then additional parsing can be done to return a properly signed URL for the storage of choice. This is not
part of the library spec here (the library API) but rather an implementation detail for this storage.
You could imagine simply implementing your own signed URLs to upload to and validate.

```python
# Split the url to only include UploadID and PartNumber parameters
parsed = urlparse(signed_url)
params = {
    x.split("=")[0]: x.split("=")[1]
    for x in parsed.query.split("&")
    if not x.startswith("X-Amz")
}
new_url = parsed.scheme + "://" + parsed.netloc + parsed.path

# Derive headers in the same way that Minio does, but include the sha256sum
signed_url = sregistry_presign_v4(
    "PUT",
    new_url,
    region=MINIO_REGION,
    content_hash_hex=sha256,
    credentials=minioExternalClient._credentials,
    expires=str(timedelta(minutes=MINIO_SIGNED_URL_EXPIRE_MINUTES).seconds),
    headers={"X-Amz-Content-Sha256": sha256},
    response_headers=params,
)
```

The successful 200 response includes the signed url (shown below).


**Response 403** 

The user is not authenticated.

**Response 404**

The upload id is not matched to the container.

**Response 200**

The signed URL was generated and is return to the client to use for the specific part.

```python
# Return the presigned url
{"data": {"presignedURL": signed_url}}
```
 
#### POST /v2/imagefile/<imageid>

This endpoint is used to push a container in the case that multipart is not supported, and should return a single URL
to upload the blob to. This can be using the same storage backend (e.g., Minio) or a custom backend implemented by the registry.
The only requirement here is to return the URL to upload to, so it's fairly flexible. After an image request is generated via `GET /v1/images/<username>/<collection>/<image>:<sha256>?arch=<arch>` the client will have an `<imageid>` to make a more formal upload request with.

**403 Response**

A 403 response is returned if the token is not valid, or the user affiliated with the token is found to not be a collection owner.

**404 Response**

A 404 response is provided if the collection in question does not exist, as looked up by name.

**200 Response**

After authentication is complete, the registry typically is going to prepare a URL for the client to upload
to storage. In practice this can mean another URL provided by the server, and also it means that we can use
the [multipart upload](https://aws.amazon.com/about-aws/whats-new/2010/11/10/Amazon-S3-Introducing-Multipart-Upload/) API
provided by AWS, and also implemented by [Minio](https://vsoch.github.io/2020/s3-minio-multipart-presigned-upload/).
At this point the registry is likely to:

 - create a storage path for the container, likely based on its unique resource identifier and digest
 - create a [presigned put object](https://docs.min.io/docs/python-client-api-reference.html#presigned_put_object) request for some bucket (or storage) with an expiration based on a registry preference.

In practice, the registry is free to implement its own upload logic, however since this API is well known
and implemented across different open source tools and vendors, it's an easy way to provide and scale storage, as the
storage can be separate from the registry API itself. At this point, the only thing that needs to be returned in the 200 response
is data with an `uploadURL`, which might look like this:

```python
{
    "data": {
        "uploadURL": "http://127.0.0.1:9000/sregistry/acollection/container%3Asha256.b059b0fdfe7ef1dcd561caa5e08c96c763df5992b8afb98ad32e1bac4d033787?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=newminio%2F20211216%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20211216T040844Z&X-Amz-Expires=300&X-Amz-SignedHeaders=host&X-Amz-Signature=c6a3c49cbed546006bf166697b13a3cfb1358b5085594bece00153a1180d2c1f"
    }
}
```

The client would then upload the container to this URL, and hit the `_complete` view upon completion.

#### PUT /v2/imagefile/imageid/_complete

This is a callback to the server to indicate that the upload is complete, and it provides the "parts" for the upload
associated with the container. The client provides
the final `upload_id` of the complete image as the only variable, along with the following body:

```python
{'uploadID': 'xxx', 'completedParts': []} # each completed part has a token which is the Etag
```
Here is an example part - the token is called an "ETag":

```python
{"partNumber":2,"token":'"684929e7fe8b996d495e7b152d34ae37-1"'}
```

This functionality in  early Singularity code can be seen [here](https://github.com/sylabs/scs-library-client/blob/30f9b6086f9764e0132935bcdb363cc872ac639d/client/push.go#L537). The registry would then generally assemble the parts as needed for the multipart upload client. For example, in Python and for Minio:

```bash
# Assemble list of parts as they are expected for Python client
parts = [
    {"ETag": x["token"].strip('"'), "PartNumber": x["partNumber"]}
    for x in body["completedParts"]
]
```

This metadata is used with the multipart upload API to complete the upload, where generally we need:

 - a bucket and specific key for the container
 - the parts
 - the Upload Id

An example with Singularity Registry Server can be seen [here](https://github.com/singularityhub/sregistry/blob/4509898d518ddbc8568700a055c66f11dcc4ff2e/shub/apps/library/views/images.py#L154).

**403 Response**

A 403 response is returned if the token is not valid, or the user affiliated with the token is found to not be a collection owner.

**404 Response**

A 404 response is provided if the container in question does not exist, as looked up by the `upload_id`.

**200 Response**

A successful response here is just the status code with empty data, e.g.,:

```python
Response(status=200, data={})
```

#### PUT /v2/imagefile/upload_id/_multipart_abort

This request can be issued at any time to abort the upload of the container. It is up to the registry to decide
on the clean up action, but generally the recommended approach is to delete partial uploads associated with the upload id and
to delete the instance of the container in the registry.

**403 Response**

The token is not valid, or the user does not have permission to act on behalf of the collection.

**404 Response**

The container does not exist.

**200 Response**

The abort operation was successful.


#### GET /v1/tags/collection_id

Get a list of tags associated with an image. This is typically called after upload to get a list of known tags.

**Response 403**

The token is not provided or authentication fails.

**Response 404**

The container collection cannot be derived from the collection id.

**Response 200**

A list of tags and associated container ids are returned for the collection.

```python
{"data": {'latest': '13'}}
```

#### POST /v1/tags/imageid

Update a list of tags for an image. This is typically called as the last step for an image push. It's the final
step to tag a newly uploaded image. The body is required to have the following data:

```python
{'Tag': 'latest', 'ImageID': '60'}
```

| Name | Description |
|------|-------------|
| Tag  | the name of the tag|
| ImageID | the image ID to apply the tag to |


Given that we find a container, we first look to see if there is an existing container with that tag.
If there is and the container is not frozen (if a registry allows it), we delete it.  
If there is an existing container and it's frozen, then we return a `400` response (see below).
Either way, we then give the container with `ImageID` the new tag.

**Response 403**

The token is not provided or authentication fails.

**Response 404**

The container cannot be found based on look up with the ImageID parameter.

**Response 400**

The container exists with the tag, and it's frozen. It cannot be replaced.

```python
{"message": "This tag exists, and is frozen."}
```

**Response 200**

The container was successfully tagged.


### Pull

Pull of an existing container comes down to the following rules:

 - containers that are public tend to be world pullable, within some rate limits that a registry can optionally apply (e.g., based on ip address or similar)
 - containers that are private must have authenticated tokens passed with the requests.
 
An example using the singularity client might look as follows:

```bash
$ singularity pull --library http://127.0.0.1 collection/container:tag
```

#### GET /v1/images/username/collection/image:tag?arch=arch

The get image view takes unique resource identifier information for a container along with an architecture,
and determines if the registry has it. 

**403 Response**

A 403 response is returned if the container is private and the token is not valid.

**404 Response**

A 404 response is provided if the collection in question does not exist, as looked up by name,
or if the architecture requested does not match the one that the container has.

**200 Response**

Given that the container is found, this view returns the same 200 success response as the similar
view pinged to upload a container defined [here](#get-image-view).

#### GET `/v1/imagefile/username/collection/image:tag?arch=arch`

This endpoint is pinged to request the actual download URL for the image.
The registry is again expected to parse the container unique resource identifier,
and respond appropriately. This typically means returning a redirect response to the correct URL to download
the image.

```python
# Retrieve the url for minio
storage = container.get_storage()
url = minioExternalClient.presigned_get_object(
    MINIO_BUCKET,
    storage,
    expires=timedelta(minutes=MINIO_SIGNED_URL_EXPIRE_MINUTES),
)
```

**404 Response**

The container matching the query was not found, or the collection is private and the user is not authenticated or doesn't have perimssion to pull.

**302 Response**

Typically, it's historically been okay to return a redirect to the URL to download.


### Notes

These are the minimal endpoints required for functionality of push and pull, both which are arguably optional for a registry
to implement. If you have any questions, please [open an issue](https://github.com/singularityhub/library-api) or look at one of the reference implementations.
If there is missing clarity in the specification please open a discussion or pull request to fix it. This is a community effort, written mostly
by @vsoch, and she can't do it all on her own (or at least hopes she doesn't have to!).

===================
OCS OpenAPI support
===================

This page explains you how you can add OpenAPI support to your app such that you can automatically generated an OpenAPI specification for your server code.

Please read the whole tutorial before starting to adjust your app.

Don't be afraid that you don't know everything from the start.
The openapi-extractor tool gives you many warnings and fails if there is something utterly broken that would not work.
Just let it run, it will tell you if there is something wrong.
Psalm will also help you validate your changes.

Requirements and prerequisites
------------------------------

- Psalm has been setup for the app
- The app supports at least Nextcloud 28
- Your APIs are exposed via OCS (see :ref:`OCS <ocscontroller>`). It is possible to also expose APIs in different ways, but those will not be explained here because they are considered legacy.

Best practices
--------------

- Be as explicit and concrete as possible about types. The closer you narrow down a type without violating any constraints the better the resulting specification will be.
- For empty data you should always use `null` if possible. If your API is well designed it should always work. If for some reason you can not do that, you should use `new \stdClass()` and not `[]`. The problem with `[]` in PHP is that will be converted into `[]` in JSON, but you want to have `{}` in JSON if the data you return is empty. For empty arrays that should be arrays it is not a problem.
- Do not throw exceptions, return valid responses with an error message instead.
- Do not use the `addHeader` method for setting headers for your responses. Right now it is not possible for psalm to trace headers you set this way, so they will not be validated by psalm.
- Try to think about what is best for an API consumer, not what is the easiest way to make it work.
- Always set all descriptions for parameters and methods. It improves the documentation and makes it easier to understand what your API does.
- Set descriptions for Controllers. Those are also included in the specification. There you can explain what the APIs that are in a given controller do or even give examples an how to connect multiple API endpoints together.

How to add OpenAPI support to your OCS API
------------------------------------------

Let's imagine you built a Todo list app for Nextcloud and have this controller:

.. code-block:: php

    class TodoApiController extends OCSController {
        #[NoAdminRequired]
        public function create(string $title, string $description = "", string $image = ""): DataResponse {
            $todo = $this->service->createTodo($title, $description, $image);

            return $this->formatTodo($todo);
        }

        #[NoAdminRequired]
        public function get(int $id): DataResponse {
            try {
                $todo = $this->service->getTodo($id);
            } catch (NotFoundException $e) {
                return new DataResponse(["error" => "Todo not found"], Http::STATUS_NOT_FOUND);
            }

            return $this->formatTodo($todo);
        }

        #[NoAdminRequired]
        public function update(int $id, string $etag, string $title = null, string $description = null, string $image = null): DataResponse {
            try {
                $todo = $this->service->updateTodo($id, $etag, $title, $description, $image);
            } catch (NotFoundException $e) {
                return new DataResponse(["error" => "Todo not found"], Http::STATUS_NOT_FOUND);
            } catch (ForbiddenException $e) {
                return new DataResponse(["error" => "ETag does not match"], Http::STATUS_BAD_REQUEST);
            }

            return $this->formatTodo($todo);
        }

        #[NoAdminRequired]
        public function delete(int $id): DataResponse {
            try {
                $todo = $this->service->deleteTodo($id);
            } catch (NotFoundException $e) {
                return new DataResponse(["error" => "Todo not found"], Http::STATUS_NOT_FOUND);
            }

            return new DataResponse(null);
        }

        private function formatTodo(Todo $todo): DataResponse() {
            return new DataResponse([
                "id" => $todo->id,
                "title" => $todo->title,
                "description" => $todo->description,
                "image" => $todo->image,
            ], Http::STATUS_OK, [
                "ETag" => $todo->etag,
            ]);
        }
    }

What you want to do now is to firstly create the correct parameter annotations and add descriptions. It could look like this:

.. code-block:: php

    /**
     * Create a new Todo
     *
     * @param string $title The title of the new Todo item
     * @param string|null $description The description of the new Todo item. Can be left empty
     * @param string|null $image The base64-encoded image of the new Todo item. Can be left empty
     */
    #[NoAdminRequired]
    public function create(string $title, string $description = null, string $image = null): DataResponse {
        ...
    }

    /**
     * Get a Todo item
     *
     * @param int $id ID of the Todo item
     */
    #[NoAdminRequired]
    public function get(int $id): DataResponse {
        ...
    }

    /**
     * Update a Todo item
     *
     * @param int $id ID of the Todo item
     * @param string $etag ETag of the Todo item. If it does not match the ETag that is stored on the server the request will be rejected
     * @param string|null $title The new title of the Todo item. Can be left empty to not update the title
     * @param string|null $description The new description of the Todo item. Can be left empty to not update the description
     * @param string|null $image The new base64-encoded image of the Todo item. Can be left empty to not update the image
     */
    #[NoAdminRequired]
    public function update(int $id, string $etag, string $title = null, string $description = null, string $image = null): DataResponse {
        ...
    }

    /**
     * Delete a Todo item
     *
     * @param int $id ID of the Todo item
     */
    #[NoAdminRequired]
    public function delete(int $id): DataResponse {
        ...
    }

The next step is to add the return types.
This is the most important step to get your API documented.
It is best to start with helper methods that are used multiple times like the `formatTodo` method in this example:

.. code-block:: php

    /**
     * @return DataResponse<Http::STATUS_OK, array{id: int, title: string, description: ?string, image: ?string}, array{ETag: string}>
     */
    private function formatTodo(Todo $todo): DataResponse() {
        ...
    }

Afterwards you can add the return types to all the other methods.
You are required to add a description for every returned status code.

.. code-block:: php

    /**
     * ...
     *
     * @return DataResponse<Http::STATUS_OK, array{id: int, title: string, description: ?string, image: ?string}, array{ETag: string}>
     *
     * 200: Todo item created
     */
    #[NoAdminRequired]
    public function create(string $title, string $description = "", string $image = ""): DataResponse {
        ...
    }

    /**
     * ...
     *
     * @return DataResponse<Http::STATUS_OK, array{id: int, title: string, description: ?string, image: ?string}, array{ETag: string}>|DataResponse<Http::STATUS_NOT_FOUND, array{error: string}, array{}>
     *
     * 200: Todo item returned
     * 404: Todo item not found
     */
    #[NoAdminRequired]
    public function get(int $id): DataResponse {
        ...
    }

    /**
     * ...
     *
     * @return DataResponse<Http::STATUS_OK, array{id: int, title: string, description: ?string, image: ?string}, array{ETag: string}>|DataResponse<Http::STATUS_BAD_REQUEST|Http::STATUS_NOT_FOUND, array{error: string}, array{}>
     *
     * 200: Todo item created
     * 400: ETag of the Todo item does not match
     * 404: Todo item not found
     */
    #[NoAdminRequired]
    public function update(int $id, string $etag, string $title = null, string $description = null, string $image = null): DataResponse {
        ...
    }

    /**
     * ...
     *
     * @return DataResponse<Http::STATUS_OK, null, array{}>|DataResponse<Http::STATUS_NOT_FOUND, array{error: string}, array{}>
     *
     * 200: Todo item deleted
     * 404: Todo item not found
     */
    #[NoAdminRequired]
    public function delete(int $id): DataResponse {
        ...
    }

How to add response definitions to share type definitions
---------------------------------------------------------

In the previous steps we have been re-using the same data structure multiple times, but it was copied every time.
This is tedious and error prone, therefore we want create some shared type definitions.
Create a new file called `ResponseDefinitions.php` in the `lib` folder of your app.
It will only work with that file name at that location.

.. code-block:: php

    /**
     * @psalm-type TodoItem = array{
     *     id: int,
     *     title: string,
     *     description: ?string,
     *     image: ?string,
     * }
     */
    class ResponseDefinitions {}

The name of every type definition has to start with the app ID.

To import and use this type definition you have to import it in your controller:

.. code-block:: php

    /**
     * @psalm-import-type TodoItem from ResponseDefinitions
     */
    class TodoApiController extends OCSController {
        ...
    }

Now you can replace every occurrence of `array{id: int, title: string, description: ?string, image: ?string}` with `TodoItem`.

How to handle exceptions
------------------------

Sometimes want to end with an exception instead of returning a response.
It is better to not do it, but when migrating existing APIs this case appears sometimes.
For this example our `update` will throw an exception when the ETag does not match:

.. code-block:: php

    #[NoAdminRequired]
    public function update(int $id, string $etag, string $title = null, string $description = null, string $image = null): DataResponse {
        ...
        } catch (ForbiddenException $e) {
            throw new OCSBadRequestException("ETag does not match");
        }
        ...
    }

Adding the annotation for that is quite simple:

.. code-block:: php

    /**
     * ...
     *
     * @throws OCSBadRequestException ETag of the Todo item does not match
     */
    #[NoAdminRequired]
    public function update(int $id, string $etag, string $title = null, string $description = null, string $image = null): DataResponse {
        ...
    }

The description after the exception class name works exactly like the description for the status codes we added earlier.
Note that the resulting response will be in plain text and no longer in JSON.
Therefore it is not recommended to use exceptions to indicate errors.

How to expose Capabilities
--------------------------

Imagine we take the same Todo app of the previous example and want to expose some capabilities to let clients know what they can expect.

.. code-block:: php

    class Capabilities implements ICapability {
        public function getCapabilities() {
            return [
                "todo" => [
                    "supported-operations" => ["create", "read", "update", "delete"],
                    "emojis-supported" => true,
                ],
            ];
        }
    }

All you have to do is add the correct return type annotation which would look like this:

.. code-block:: php

    class Capabilities implements ICapability {
        /**
         * @return array{todo: array{supported-operation: string[], emojis-supported: bool}}
         */
        public function getCapabilities() {
            return [
                "todo" => [
                    "supported-operations" => ["create", "read", "update", "delete"],
                    "emojis-supported" => true,
                ],
            ];
        }
    }

It will automatically appear in the generated specification.

How to generate the specification
---------------------------------

The specification is generated by the `openapi-extractor <https://github.com/nextcloud/openapi-extractor>`_.
Run the `generate-spec` script inside the root folder of your app and if you did everything right you will have a new file called `openapi.json`.
If it fails somewhere it will tell you what is wrong and often times also how to fix it.
Additionally you should run psalm to check for any problems.

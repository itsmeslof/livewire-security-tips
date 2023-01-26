This repo mostly serves as a reminder to myself the next time I work on a Livewire project and get clumsly :^)

***tl;dr:*** public properties on a Livewire component can be manipulated from the frontend, causing the component to re-render and potentially re-query access-restricted data using the updated property.

- Note: this only applies to Livewire versions < Livewire 3 (not yet released). Livewire v3 will introduce locked properties:
~~~php
/** @locked */
public $postId; // Will throw an error if it is changed from the frontend
~~~
More information: https://youtu.be/f4QShF42c6E?t=6947

-------

### Consider the following situation where a non-admin user is viewing a blog post they made, and you have a Livewire component to show it:

~~~php
<?php
// ShowPost livewire component

class ShowPost extends Component
{
    // This is PUBLIC on the frontend and can be changed
    // by the user, even if we DON'T use wire:model.
    public $postId;

    // Consider this the equivalent to a controller action,
    // except a controller action is called on *every* request,
    // unlike mount() which is only called on the *initial* request.
    // Therefore we can't simply authorize here the same way we would
    // in a controller action.
    public function mount($postId)
    {
        $this->postId = $postId;
        
        // We authorize on the initial load, but mount() will 
        // NOT be called on subsequent requests to update public properties.
        $this->authorize('view', Post::find($postId));
    }

    public function render()
    {
        // Any time $postId is changed this will re-render with
        // the queried Post, which may not belong to the user.

        // Also, you shouldn't pass an entire model to the frontend
        // this way, as you may be exposing sensitive data. Consider a
        // DTO, local public array/object, or public properties with only
        // the data you want to expose.
        return view('livewire.posts.show', [
            'post' => Post::find($this->postId)
        ]);
    }
}
~~~

If the user runs the following in their devtools console:

~~~js
Livewire.first().postId = 'another-users-post-id-they-shouldnt-be-able-to-view';
~~~

it will update the `$postId` property on the Livewire component, but it will NOT re-authorize via mount(), because mount() will not be called again, and instead it will re-render with the post data from another user. This differs from a normal Laravel route and controller action, because in that case you would have something like:

`GET /posts/{post:id}` - which calls the following controller method ***every time the $postId value is changed in the url***:

~~~php
// With a normal route and controller, it is *impossible* for the user to
// manipulate the value without authorization being checked.
public function show(Post $post)
{
    $this->authorize('view', $post);

    return view('posts.show', ['post' => $post]);
}
~~~

Because of this, with a Livewire component you need to take extra measures to ensure you're authorizing access in ***EVERY*** situation in which queried data may be changed from the frontend, whether it's from `wire:model` or data manipulation.

**Wait, why can't this data just be private?** Livewire components do not have state on the server. Each request passes all the data that represents the current state to the request and then receives it in the response. Because of the lack of state on the server, `protected` and `private` properties do not persist between Livewire updates and should not be used to store state. Because of this, anything you need to show the user (such as showing them the details of the Post model), you need to make that property `public`. See more: (https://laravel-livewire.com/docs/2.x/properties#important-notes).

So what can you do to prevent users from manipulating properties on the frontend and viewing data that doesn't belong to them? To ensure access-restricted data is not being queried in the event a user manipulates a public property that isn't designed to be changed, you can listen to either of the following component lifecycle hooks, re-query and re-authorize before `render` is called:

- Listen to the `updated` hook on your Livewire component:
    - This hook "Runs after any update to the Livewire component's data (Using `wire:model`, not directly inside PHP)".

- Listen to the `updatedFoo` hook on your Livewire component:
    - This hook "Runs after a property called `$foo` is updated. Array properties have additional `$key` argument as above".
    - This can be useful if you have a lot of public properties, but only specific ones that are used to query data. Listening to this instead of the generic `updated` hook can save you from doing unnecessary queries.

More details on lifecycle hooks: https://laravel-livewire.com/docs/2.x/lifecycle-hooks.

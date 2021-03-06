### How SilverStripe handles forms

This lesson is about frontend forms, which is a very important distinction to make, because we've actually been working with forms since very early on in the CMS. Whenever we're editing data in the backend, we're using a form. The main difference is that in the CMS, only a small part of that form is exposed to the model class via `getCMSFields()`.

When you think about how a framework might deal with form creation, you may imagine a simple object that takes an array of form fields and ultimately dumps some HTML via a `render()` method or something similar, but one of the most distinguishing features of forms in SilverStripe is that they are treated a _first-class citizens_, which is to say they're intelligent objects that work with all the big players in the request cycle.

#### Creating a simple form

Let's look at a simple form constructor to better understand how this will work. The following method would be placed in any `Controller`.

```php
    public function ContactForm()
    {
        $myForm = Form::create(
            $controller,
            'ContactForm',
            FieldList::create(
                TextField::create('YourName','Your name'),
                TextareaField::create('YourComments','Your comments')
            ),
            FieldList::create(
                FormAction::create('sendContactForm','Submit')
            ),
            RequiredFields::create('YourName','YourComments')
        );

        return $myForm;
    }
```

Let's walk through each argument of the constructor:

*   `$controller` A form should be generated by, and handled by, a Controller. The benefits to this are impressive, and we'll see some of them in a bit. Because the controller that creates the form usually handles its submission as well, 99% of the time, this first argument is going to be `$this`.
*   `ContactForm` A string representing the _name of the method_ that creates the form. Odd, right? It's a bit unconventional to be dealing with class methods as strings, for sure, but this is a really key feature. The form will submit to a URL that calls this method, which means that we get the fully configured `Form` object to inspect and manipulate when handling submission. We can add error messages, set state, and even manipulate the fields based on the user's submission. Because this argument always describes the name of the function we're in, you can simply use the PHP constant `__FUNCTION__` here.
*   A `FieldList` object containing all of the form fields that accept user input
*   A `FieldList` object that contains all the form _actions_, or buttons, that submit a form. Often times you'll only have one action. The first argument of the `FormAction` constructor is the method on the controller that will be invoked when the form is submitted.
*   A `RequiredFields` object takes a list of form field names to validate. Validation, as you may know, is infinitely complex. In this case, we're doing very basic validation that simply ensures that the fields contain valid data. Each form field validates itself, so an `EmailField` will validate differently than a simple `TextField`.

#### Handling submission

In the `FieldList` of actions for our form, we specified a single form action that maps to the method `sendContactForm`. Let's look at what that method might do. Again, the following code could be in any `Controller` class.

```php
    public function ContactForm()
     {
        //...
    }

    public function sendContactForm($data, $form)
    {
        $name = $data['YourName'];
        $message = $data['YourMessage'];
        if(strlen($message) < 10) {
            $form->addErrorMessage('YourMessage','Your message is too short','bad');
            return $this->redirectBack();
        }

        return $this->redirect('/some/success/url');
    }
```

Our form handler method is given two parameters:

*   `$data` contains an array of all the form field names mapped to their values, very similar to what you would get from `$_POST`.
*   `$form` is the form object that the `ContactForm` method gave us. Remember that we had to specify the name of the function in our form constructor? This is why. Now our form handler has full access to that form, which is immeasurably useful.

So that's the really high-level view of how forms work. Now we'll look at implementing a working frontend form in our project.

### Adding a form to a template

Let's look at our `ArticlePage.ss` template again and find the comment form near the bottom of the page. Let's try our best to replicate that form in our controller.

_app/src/ArticlePageController.php_
```php
//...
use SilverStripe\Forms\Form;
use SilverStripe\Forms\FieldList;
use SilverStripe\Forms\TextField;
use SilverStripe\Forms\EmailField;
use SilverStripe\Forms\TextareaField;
use SilverStripe\Forms\FormAction;
use SilverStripe\Forms\RequiredFields;

class ArticlePageController extends PageController
{

    public function CommentForm()
    {
        $form = Form::create(
            $this,
            __FUNCTION__,
            FieldList::create(
                TextField::create('Name',''),
                EmailField::create('Email',''),
                TextareaField::create('Comment','')
            ),
            FieldList::create(
                FormAction::create('handleComment','Post Comment')
            ),
            RequiredFields::create('Name','Email','Comment')
        );

        return $form;
    }
}
```

This is all very similar to what we did in the example. Notice we're leaving the labels for the fields deliberately blank. That's because in the design, they're added with `placeholder` attributes. Let's make a small update to populate the placeholders of the form fields.

_app/src/ArticlePage.php_
```php
    public function CommentForm()
    {
        //...
            FieldList::create(
                TextField::create('Name','')
                    ->setAttribute('placeholder','Name*'),
                EmailField::create('Email','')
                    ->setAttribute('placeholder','Email*'),
                TextareaField::create('Comment','')
                    ->setAttribute('placeholder','Comment*')
            ),
        //...
    }
```

Since this is a public method on the controller, we can add it to the template by calling `$CommentForm`. Add that variable in place of the static form markup. Refresh the page and have a look.

Looks pretty awful, right? There are still a number of modifications we have to make to our form to keep it in line with the markup the designer provided. Let's make a few more updates.
```php
public function CommentForm()
{
    $form = Form::create(
        $this,
        __FUNCTION__,
        FieldList::create(
            TextField::create('Name','')
                ->setAttribute('placeholder','Name*')
                ->addExtraClass('form-control'),
            EmailField::create('Email','')
                ->setAttribute('placeholder','Email*')
                ->addExtraClass('form-control'),
            TextareaField::create('Comment','')
                ->setAttribute('placeholder','Comment*')
                ->addExtraClass('form-control')
        ),
        FieldList::create(
            FormAction::create('handleComment','Post Comment')
                ->setUseButtonTag(true)
                ->addExtraClass('btn btn-default-color btn-lg')
        ),
        RequiredFields::create('Name','Email','Comment')
    );

    $form->addExtraClass('form-style');

    return $form;
}
```

We use the chainable methods `setAttribute` and `addExtraClass` for the main fields and for the form itself, and `setUseButtonTag` for the form action to force it to render a `<button>` instead of an `<input>`. Refresh, and things should look a lot better now.

This is getting a bit verbose. Let's see if we can tighten all that up to add the extra classes and placeholders dynamically.
```php
public function CommentForm()
{
    $form = Form::create(
        $this,
        __FUNCTION__,
        FieldList::create(
            TextField::create('Name',''),
            EmailField::create('Email',''),
            TextareaField::create('Comment','')
        ),
        FieldList::create(
            FormAction::create('handleComment','Post Comment')
                ->setUseButtonTag(true)
                ->addExtraClass('btn btn-default-color btn-lg')
        ),
        RequiredFields::create('Name','Email','Comment')
    )
    ->addExtraClass('form-style');

    foreach($form->Fields() as $field) {
        $field->addExtraClass('form-control')
               ->setAttribute('placeholder', $field->getName().'*');            
    }

    return $form;
}
```

Form methods are chainable, just like form field methods, so we've chained `addExtraClass` to the constructor. We use the `Fields()` method of the form to get each field by reference and add the extra class to each one, and create a dynamic placeholder based on the field's name.

This is mostly just a demonstration of form methods.. You probably wouldn't want to do something this aggressive in your project code because it doesn't scale well. Imagine if we had a new field that wasn't required, for instance, and the placeholder shouldn't have an asterisk next to it. Sometimes, it's okay to be verbose and repeat yourself a little bit. You'll be glad you did when you have to make small updates and handle edge cases.

If you're a bit dismayed about having to manually add all of the basic requirements for Bootstrap, rest assured there is a `bootstrap-forms` module that automatically does most of these updates. We haven't talked about modules yet, so it's a bit out of scope for now, but be aware that this is a bit more complex than it needs to be.

### Creating a Comment data model

When a user submits a comment, we want to save it to the database and add it to the page. Before we go any further with forms, we're going to need to do some data modeling to store all that content.

Let's create a simple `ArticleComment` DataObject. We've seen all this before. _app/src/ArticleComment.php_

```php
<?php

namespace SilverStripe\Lessons;

use SilverStripe\ORM\DataObject;

class ArticleComment extends DataObject
{

    private static $db = [
        'Name' => 'Varchar',
        'Email' => 'Varchar',
        'Comment' => 'Text'
    ];

    private static $has_one = [
        'ArticlePage' => ArticlePage::class,
    ];
}
```

Notice we have a `$has_one` back to `ArticlePage` to set up a `$has_many` relationship. Let's now follow through with that.

_app/src/ArticlePage.php_
```php
class ArticlePage extends Page
{

    //...
    private static $has_many = [
        'Comments' => ArticleComment::class,
    ];
    //...
}
```

Run `dev/build` and see that you get a new table.

While we're in here, let's carve up the template and add a loop for all the comments. There are no comments right now, but there will be shortly.

_app/templates/Layout/ArticlePage.ss_ (line 72)
```html
<div class="comments">
    <ul>
        <% loop $Comments %>                        
        <li>
            <img src="images/comment-man.jpg" alt="" />
            <div class="comment">                                
                <h3>$Name<small>$Created.Format('j F, Y')</small></h3>
                <p>$Comment</p>
            </div>
        </li>
        <% end_loop %>
    </ul>
```
Refresh, and the expected result is that no comments appear above the form.

### Handling form submission

Now that we have our data models set up, we can start using them in our form handler. Looking at our form method, the name of the handler we've specified is `handleComment()`. Let's create that method, right below the form creation method.

_app/src/ArticlePageController.php_
```php
public function CommentForm()
{
    //...
}

public function handleComment($data, $form)
{
    $comment = ArticleComment::create();
    $comment->Name = $data['Name'];
    $comment->Email = $data['Email'];
    $comment->Comment = $data['Comment'];
    $comment->ArticlePageID = $this->ID;
    $comment->write();

    $form->sessionMessage('Thanks for your comment!','good');

    return $this->redirectBack();
}
```

In the handler, we optimistically create the `ArticleComment` object as a first operation. We can do this because at this point the form has already passed validation, so we know that all of the required fields are in the `$data` array. You may not always want to do this. You might have some logic that determines otherwise based on the values provided, but let's just keep it simple for now.

Notice that we create that `$has_many` relation by binding the comment back to the `ArticlePage`. That's an easy step to forget. Remember that `$has_one` fields are always suffixed with `ID`. `$this->ID` in this case refers to the current page ID. All the properties of a page (`Title`, `Content`, `ID`, etc.) are available in the controller as well as the model thanks to SilverStripe's `SilverStripe\CMS\Controllers\ModelAsController` class.

Let's try submitting the form and see what happens.

```
    Action 'CommentForm' isn't allowed on class SilverStripe\Lessons\ArticlePageController.
```

Yikes!

What's happening here? We'll get to the error in just a moment, but right now, it's important to understand where we are and what we're looking at.

Take a look at the URL. `travel-guides/sample-article-1/CommentForm`. The first two segments of that are easily identifiable. It's the URL of our current article page. The last part, `CommentForm` is what's called a _controller action_. By default, the URL part that immediately follows the page URL will tell the controller to invoke a method by that name. In this case, we want the `CommentForm()` method on our controller to execute, because it creates the `Form` object, which is then passed along to our form submission handler. This, right here, is where most of the magic of forms happens in SilverStripe. They actually submit to a URL that recreates them as they were rendered to the user.

You may have noticed that I casually mentioned an an alarming detail of request handling in SilverStripe -- **you can invoke arbitrary methods in the URL**.

Exhale. It's not that simple.

In fact, that's precisely why we're seeing this error. We can't just execute arbitrary controller methods from the URL. The method has to be whitelisted using a static variable known as `$allowed_actions`. Let's do that now.

_app/src/ArticlePage.php_
```php
//...
class ArticlePageController extends PageController {

    private static $allowed_actions = [
        'CommentForm',
    ];

    //...
}
```

We made a change to a static variable, so we have to run `?flush`. Go back to the article page (i.e. remove **/CommentForm/** from the URL) and run the flush. We don't want to do this in the middle of a form submission.

Try submitting the form again. You should now see your comment posted.

#### Binding to a DataObject

Let's take this a step further. We've talked about how forms are first-class citizens in SilverStripe. Part of that is being model-aware. Looking at our handler method, we see that all the form parameters are named exactly the same as the `ArticleComment` database fields. This is ideal, because it means we can take advantage of a massive time-saving method of the form class known as `saveInto()`.

Let's modify our function to call `saveInto()` instead of manually assigning all the form values.

_app/src/ArticlePageController.php_
```php
    public function handleComment($data, $form)
    {
        $comment = ArticleComment::create();
        $comment->ArticlePageID = $this->ID;
        $form->saveInto($comment);
        $comment->write();

        $form->sessionMessage('Thanks for your comment','good');

        return $this->redirectBack();
    }
```

Notice that we still have to manually assign the `ArticlePageID` field, as that is not present in the form data. We could have passed it via a hidden input, which would eliminate that line of code.

What's great about this method is that it can actually respond to the needs of a specific model. If our `ArticleComment` object had a method called `saveComment()` or `saveName()`, it could save the form data in its own specific way. So it may look like a shotgun approach, but it can actually be pretty granular if you want it to be.

### Dealing with validation

Our form is accepting submissions and working as expected, so let's now add a bit of validation. We're already using `RequiredFields`, which is our primary sentinel against bad data, but what if we want to add some custom logic that goes beyond simple sanity checks?

#### Adding custom validation logic to the handler

If the logic were really complicated, we could write our own validator, which we'll cover in the future, but for simple validation, it's fine to do all of this in your form handler method. Let's run a check to make sure the user's comment has not already been added. You might think of this as really basic spam protection.

_app/src/ArticlePage.php_
```php
    public function handleComment($data, $form)
    {
        $existing = $this->Comments()->filter([
            'Comment' => $data['Comment']
        ]);
        if($existing->exists() && strlen($data['Comment']) > 20) {
            $form->sessionMessage('That comment already exists! Spammer!','bad');

            return $this->redirectBack();
        }        

        // ...
    }
```

Before creating the `Comment` object, we first inspect the `$data` array to see if everything looks right. We look for a comment on this page specifically that contains the same content, and if so, we add a message to the top of the form. The value `'bad'` as the second argument gives it an appropriate CSS class. `'good'` is the other option here.

To filter out false positives, we make sure the comment is at least 20 characters long. It's plausible that multiple readers might comment "Nice article" or "Good work" and we don't want to punish them.

Again, the lesson here is not about spam protection, but just how to handle form validation. Don't accept this as gospel on how to secure your blog comments.

Try submitting the form again with an existing comment, and you'll see that we generate an error message.

#### Preserving state

There's one usability problem here, and perhaps we shouldn't worry about it too much since we're not particularly motivated to be nice to spammers, but for the sake of teaching the concept, it would be nice if the form saved its invalid state, so that the user doesn't have to repopulate an empty form. For this, the convention is to use `Session` state.

_app/src/ArticlePageController.php_
```php
    public function handleComment($data, $form)
    {
        $session = $this->getRequest()->getSession();
        $session->set("FormData.{$form->getName()}.data", $data);
        //...
        $comment->write();

        $session->clear("FormData.{$form->getName()}.data");
        $form->sessionMessage('Thanks for your comment!','good');

        return $this->redirectBack();
    }
```

We create a SKU using the form name to use as a session token, and store the `$data` array there. If everything checks out, we clear it, so that the form renders clean on the next page load. If not, we're going to want the form to render the session data.

_app/src/ArticlePageController.php_
```php
    public function CommentForm()
    {
        //...

        foreach($form->Fields() as $field) {
            $field->addExtraClass('form-control')
                  ->setAttribute('placeholder', $field->getName().'*');            
        }

        $data = $this->getRequest()->getSession()->get("FormData.{$form->getName()}.data");

        return $data ? $form->loadDataFrom($data) : $form;
    }
```

Using the ternary operator, we look to see if `$data` exists. If it does, return the form with the data loaded into it using `loadDataFrom`. If not, just return the form as is. Remember, most form methods all return the `Form` object, so it's okay to return the result of `loadDataFrom()` here.

Test out your form and see that it now preserves its state after failed validation.

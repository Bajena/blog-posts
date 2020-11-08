
---
title: Hanami: Unobtrusive JavaScript
published: false
description:
tags: hanami ruby
---

# Intro
In [one](https://dev.to/bajena/hanami-thoughts-of-a-rails-developer-ik1) of my previous posts I wrote down the impressions about Hanami I've had while working on my side project, [Flashcard Genius](https://flashcard-genius.com). One of the things I mentioned there was that some features you'd expect from a framework are missing or underdocumented. One of such features I missed in Hanami was unobtrusive javascript.

Rails comes out-of-the-box with [helpers](https://guides.rubyonrails.org/working_with_javascript_in_rails.html) that allow  developers to do following things without writing a single JavaScript line:
- Create links to POST/DELETE/PATCH endpoints (`link_to "Delete article, @article, method: :delete`)
- Send forms remotely using AJAX just by adding `remote: true` to your form tag.
- Show simple confirmation dialog after clicking a link (`link_to "Dangerous zone", dangerous_zone_path, data: { confirm: 'Are you sure?' }`)
- Disable submit buttons while waiting for the form to be sent (`f.submit data: { "disable-with": "Saving..." }`)

In my application I had a need for remote DELETE links (removing flashcards, log out endpoint) and remote forms (quickly add/edit a card without reloading the page), so I started looking for solutions and... turns out there's a way to do it in Hanami thanks to a plugin called [hanami-ujs](https://github.com/hanami/ujs). Surprisingly, this library is hosted in the official Hanami GitHub organization, but the documentation almost doesn't exist. In this post I'm gonna show you how I solved some of my problems using this library.

# Installation
Installation and setup is well described in gem's [README](https://github.com/hanami/ujs) file, have a look there.

# Use case #1 - DELETE links
When you're following RESTful resource routing then there's a 99% chance that a route to destroy a resource will be a `DELETE` method route. Unfortunately Hanami doesn't support creating links to DELETE paths out-of-the box, so e.g. my logout link looked like this under the hood:
```ruby
<%=
  form_for :session, routes.logout_path, method: :delete do
    a href: "#", onclick: "this.closest('form').submit();return false;" do
      "Logout"
    end
  end
%>
```

Even though it worked it was a nasty hack. It was also violating [`unsafe-inline` directive](https://content-security-policy.com/unsafe-inline/) from Content Security Policy.

After installing `hanami-ujs` this is one line is all I need instead:
```ruby
<%= link_to "Logout", routes.logout_path, "data-method": :delete %>
```

# Use case #2 - Remote forms
Sometimes you want your view to be a bit more dynamic and don't want to reload the whole page when a form is sent. In this case AJAX is your friend. `hanami-ujs` offers sending remote AJAX forms. I used this feature e.g. on the word edit form in Flashcard Genius. You can see how that works on the GIF below:

![screencast 2020-11-08 12-37-52](https://user-images.githubusercontent.com/5732023/98463922-61e80d80-21bf-11eb-937d-9711b63a40c9.gif)

In order to achieve this result I needed two things - form definition and JS event handlers.

## Form definition

If you know a bit of Hanami you'll notice that this is a standard Hanami form with two additions:
- `remote: true` - tells `hanami-ujs` that the form will be sent by AJAX
- `"data-method": :patch` option - method that `hanami-ujs` will use to send the form

```ruby
<%=
  form_for(:word, routes.word_path(word.id), remote: true, "data-method": :patch, class: "edit-word-form") do
    div class: "card-body" do
      div class: "row margin-bottom-none" do
        div class: "col xs-12" do
          div class: "form-group" do
            label      :question
            text_field :question, class: "input-block", required: true, value: word.question
          end
        end

        div class: "col xs-12" do
          div class: "form-group" do
            label      :question_example
            text_field :question_example, class: "input-block", value: word.question_example
          end
        end

        div class: "col xs-12" do
          div class: "form-group" do
            label      :answer
            text_field :answer, class: "input-block", required: true, value: word.answer
          end
        end

        div class: "col xs-12" do
          div class: "form-group" do
            label      :answer_example
            text_field :answer_example, class: "input-block", value: word.answer_example
          end
        end
      end
    end

    div class: "card-footer" do
      div class: "row margin-none padding-none flex-edges" do
        span("Cancel", class: "paper-btn btn-small btn-primary-outline cancel-edit-button margin-none")
        submit("Save", class: "btn-secondary-outline btn-small edit-save-button margin-none", "data-disable-with": "Saving...")
      end
    end
  end
%>

```

##  JS event handler
In order to be able to dynamically react on AJAX response we need some JavaScript on our page. `hanami-ujs` defines two event types:
- `ajax:before` - Called before the request is sent. It can be used e.g. to clear forms errors after previous request.
- `ajax:complete` - Called when the response is received (no matter if the request succeeded or not).

Here's how my JS code for the edit word form looks like.  The server updates the word and returns a HTML template with the updated record. The new HTML replaces the old one:
```javascript
function updateCompleteHandler(event) {
  var form = event.target;

  // There can be other `ajax:complete` handlers defined on the page.
  // Make sure that this code is executed when `edit-word-form` is sent.
  if (!form.className.includes("edit-word-form")) {
    return;
  }

  var status = event.detail.status;

  if (status >= 200 && status < 300) {
    alertify.success("Word updated");

    var card = form.closest(".flashcard-column");
    // Replace the current word with server's response
    card.outerHTML = event.detail.response;
  } else {
    alertify.error("Update failed");

    var saveButton = form.getElementsByClassName("edit-save-button")[0];
    saveButton.disabled = false;
    saveButton.innerText = "Save"
  }
}

document.addEventListener("ajax:complete", updateCompleteHandler);
```

# Additional useful features
`hanami-ujs` has also two other useful features:
- You can add `"disable-with": "Saving..."` to your form submit buttons. This'll automatically disable the button to prevent double-sends and replace the its text with `Saving...`.
One drawback of this feature is that it wasn't created with AJAX forms in mind, so when you want to it `disable-with` with remote forms you'll have to re-enable the button and change its text back to original after the request. You can find the example in previous snippets.
- You can show a simple confirmation dialog that prevents sending a form/following a link by adding `"data-confirm": "Are you sure you want to do it?"`.
This is how one of the buttons in my app looks like:
  ```ruby
  <%=
    link_to(
      "",
      routes.word_list_path(word_list.id),
      id: "delete-list-button",
      class: "paper-btn btn-danger-outline btn-small margin-none fas fa-trash-alt",
      "data-method": :delete,
      "data-confirm": "Are you sure you want to delete this list?",
      "data-disable-with": "",
      title: "Delete list"
    )
  %>
    ```

  which results in a following message:

  ![image](https://user-images.githubusercontent.com/5732023/98468076-1b9fa800-21d9-11eb-983a-9131bb3b2438.png)

# Summary
Even though `hanami-ujs` may seem quite abandoned it provides tools that save a lot of time when creating simple server-rendered apps. If you're creating such an app in Hanami there's a big chance you'll want to use it. For more `hanami-ujs` examples (and Hanami in general) you can check Flashcard Genius' repository at https://github.com/Bajena/flashcard-genius.

Happy coding :)

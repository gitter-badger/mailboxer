# Mailboxer [![Build Status](https://travis-ci.org/chriscz/mailboxer.svg?branch=master)](https://travis-ci.org/chriscz/mailboxer) [![Gem Version](https://badge.fury.io/rb/mailboxer.png)](http://badge.fury.io/rb/mailboxer)

This project is based on the need for a private message system for [ging / social\_stream](https://github.com/ging/social_stream). Instead of creating our core message system heavily
dependent on our development, we are trying to implement a generic and
potent messaging gem.

After looking for a good gem to use we noticed the lack of messaging gems
and functionality in them. Mailboxer tries to fill this void delivering
a powerful and flexible message system. It supports the use of
conversations with two or more participants, sending notifications to
recipients (intended to be used as system notifications “Your picture has
new comments”, “John Doe has updated his document”, etc.), and emailing the
messageable model (if configured to do so). It has a complete implementation
of a `Mailbox` object for each messageable with `inbox`, `sentbox` and
`trash`.

The gem is constantly growing and improving its functionality. As it is
used with our parallel development [ging / social\_stream](https://github.com/ging/social_stream) we are finding and fixing bugs continously. If you want
some functionality not supported yet or marked as TODO, you can create
an [issue](https://github.com/chriscz/mailboxer/issues) to ask for it. It will be great feedback for us, and we
will know what you may find useful in the gem.

Mailboxer was born from the great, but outdated, code from [lpsergi /
acts_as_messageable](https://github.com/psergi/acts_as_messageable).

We are now working to make exhaustive documentation and some wiki
pages in order to make it even easier to use the gem to its full potential.
Please, give us some time if you find something missing or [ask for
it](https://github.com/chriscz/mailboxer/issues).  You can also find us on the [Gitter room for this repo](https://gitter.im/chriscz/mailboxer).  Join us there to talk.

Installation
------------

Add to your Gemfile:

```ruby
gem 'mailboxer'
```

Then run:

```sh
$ bundle install
```

Run install script:

```sh
$ rails g mailboxer:install
```

And don't forget to migrate your database:

```sh
$ rake db:migrate
```

You can also generate email views:

```sh
$ rails g mailboxer:views
```

Upgrading
---------

If upgrading from 0.11.0 to 0.12.0, run the following generators:

```sh
$ rails generate mailboxer:namespacing_compatibility
$ rails generate mailboxer:install -s
```

Then, migrate your database:

```sh
$ rake db:migrate
```

## Requirements & Settings

### Emails

We are now adding support for sending emails when a Notification or a Message is sent to one or more recipients. You should modify the mailboxer initializer (/config/initializer/mailboxer.rb) to edit these settings:

```ruby
Mailboxer.setup do |config|
  #Enables or disables email sending for Notifications and Messages
  config.uses_emails = true
  #Configures the default `from` address for the email sent for Messages and Notifications of Mailboxer
  config.default_from = "no-reply@dit.upm.es"
  ...
end
```

You can change the way in which emails are delivered by specifying a custom implementation of notification and message mailers:

```ruby
Mailboxer.setup do |config|
  config.notification_mailer = CustomNotificationMailer
  config.message_mailer = CustomMessageMailer
  ...
end
```

If you have subclassed the Mailboxer::Notification class, you can specify the mailers using a member method:

```ruby
class NewDocumentNotification < Mailboxer::Notification
  def mailer_class
    NewDocumentNotificationMailer
  end
end

class NewCommentNotification < Mailboxer::Notification
  def mailer_class
    NewDocumentNotificationMailer
  end
end
```

Otherwise, the mailer class will be determined by appending 'Mailer' to the mailable class name.

### User identities

Users must have an identity defined by a `name` and an `email`. We must ensure that Messageable models have some specific methods. These methods are:

```ruby
#Returning any kind of identification you want for the model
def name
  return "You should add method :name in your Messageable model"
end
```

```ruby
#Returning the email address of the model if an email should be sent for this object (Message or Notification).
#If no mail has to be sent, return nil.
def mailboxer_email(object)
  #Check if an email should be sent for that object
  #if true
  return "define_email@on_your.model"
  #if false
  #return nil
end
```

These names are explicit enough to avoid colliding with other methods, but as long as you need to change them you can do it by using mailboxer initializer (/config/initializer/mailboxer.rb). Just add or uncomment the following lines:

```ruby
Mailboxer.setup do |config|
  # ...
  #Configures the methods needed by mailboxer
  config.email_method = :mailboxer_email
  config.name_method = :name
  config.notify_method = :notify
  # ...
end
```

You may change whatever you want or need. For example:

```ruby
config.email_method = :notification_email
config.name_method = :display_name
config.notify_method = :notify_mailboxer
```

Will use the method `notification_email(object)` instead of `mailboxer_email(object)`, `display_name` for `name` and `notify_mailboxer` for `notify`.

Using default or custom method names, if your model doesn't implement them, Mailboxer will use dummy methods so as to notify you of missing methods rather than crashing.

## Preparing your models

In your model:

```ruby
class User < ActiveRecord::Base
  acts_as_messageable
end
```

You are not limited to the User model. You can use Mailboxer in any other model and use it in several different models. If you have ducks and cylons in your application and you want to exchange messages as if they were the same, just add `acts_as_messageable` to each one and you will be able to send duck-duck, duck-cylon, cylon-duck and cylon-cylon messages. Of course, you can extend it for as many classes as you need.

Example:

```ruby
class Duck < ActiveRecord::Base
  acts_as_messageable
end
```

```ruby
class Cylon < ActiveRecord::Base
  acts_as_messageable
end
```

## Mailboxer API

### Warning for version 0.8.0
Version 0.8.0 sees `Messageable#read` and `Messageable#unread` renamed to `mark_as_(un)read`, and `Receipt#read` and `Receipt#unread` to `is_(un)read`. This may break existing applications, but `read` is a reserved name for Active Record, and the best practice in this case is simply avoid using it.

### How can I send a message?

```ruby
#alfa wants to send a message to beta
alfa.send_message(beta, "Body", "subject")
```

### How can I read the messages of a conversation?

As a messageable, what you receive are receipts, which are associated with the message itself. You should retrieve your receipts for the conversation and get the message associated with them.

This is done this way because receipts save the information about the relation between messageable and the messages: is it read?, is it trashed?, etc.

```ruby
#alfa gets the last conversation (chronologically, the first in the inbox)
conversation = alfa.mailbox.inbox.first

#alfa gets it receipts chronologically ordered.
receipts = conversation.receipts_for alfa

#using the receipts (i.e. in the view)
receipts.each do |receipt|
  ...
  message = receipt.message
  read = receipt.is_unread? #or message.is_unread?(alfa)
  ...
end
```

### How can I reply to a message?

```ruby
#alfa wants to reply to all in a conversation
#using a receipt
alfa.reply_to_all(receipt, "Reply body")

#using a conversation
alfa.reply_to_conversation(conversation, "Reply body")
```

```ruby
#alfa wants to reply to the sender of a message (and ONLY the sender)
#using a receipt
alfa.reply_to_sender(receipt, "Reply body")
```

### How can I delete a message from trash?

```ruby
#delete conversations forever for one receipt (still in database)
receipt.mark_as_deleted

#you can mark conversation as deleted for one participant
conversation.mark_as_deleted participant

#Mark the object as deleted for messageable
#Object can be:
  #* A Receipt
  #* A Conversation
  #* A Notification
  #* A Message
  #* An array with any of them
alfa.mark_as_deleted conversation

# get available message for specific user
conversation.messages_for(alfa)
```
### How can I retrieve my conversations?

```ruby
#alfa wants to retrieve all his conversations
alfa.mailbox.conversations

#A wants to retrieve his inbox
alfa.mailbox.inbox

#A wants to retrieve his sent conversations
alfa.mailbox.sentbox

#alfa wants to retrieve his trashed conversations
alfa.mailbox.trash

#Also wants to retrieve not trashed conversations
alfa.mailbox.conversations({ :mailbox_type => 'not_trash' })

```

### How can I paginate conversations?

You can use Kaminari to paginate the conversations as normal. Please, make sure you use the last version as mailboxer uses `select('DISTINCT conversations.*')` which was not respected before Kaminari 0.12.4 according to its changelog. Working correctly on Kaminari 0.13.0.

```ruby
#Paginating all conversations using :page parameter and 9 per page
conversations = alfa.mailbox.conversations.page(params[:page]).per(9)

#Paginating received conversations using :page parameter and 9 per page
conversations = alfa.mailbox.inbox.page(params[:page]).per(9)

#Paginating sent conversations using :page parameter and 9 per page
conversations = alfa.mailbox.sentbox.page(params[:page]).per(9)

#Paginating trashed conversations using :page parameter and 9 per page
conversations = alfa.mailbox.trash.page(params[:page]).per(9)
```

You can take a look at the full documentation for Mailboxer in [rubydoc.info](http://rubydoc.info/gems/mailboxer/frames).

## Do you want to test Mailboxer?

Thanks to [Roman Kushnir (@RKushnir)](https://github.com/RKushnir/) you can test Mailboxer with this [sample app](https://github.com/RKushnir/mailboxer-app).

## I need a GUI!

If you need a GUI you should take a look at these links:

* The wiki page [GUI Example on a real application](https://github.com/chriscz/mailboxer/wiki/GUI-Example-on-a-real-application).

## Merged Pull Requests
* https://github.com/mailboxer/mailboxer/pull/508
* https://github.com/mailboxer/mailboxer/pull/507
* https://github.com/mailboxer/mailboxer/pull/504
* https://github.com/mailboxer/mailboxer/pull/503
* https://github.com/mailboxer/mailboxer/pull/497
* https://github.com/mailboxer/mailboxer/pull/409
* https://github.com/mailboxer/mailboxer/pull/405
* https://github.com/mailboxer/mailboxer/pull/396
* https://github.com/mailboxer/mailboxer/pull/515

## Contributors
* [Roendal](https://github.com/chriscz/mailboxer/commits/master?author=Roendal) (Eduardo Casanova)
* [dickeyxxx](https://github.com/chriscz/mailboxer/commits/master?author=dickeyxxx) (Jeff Dickey)
* [RKushnir](https://github.com/chriscz/mailboxer/commits/master?author=RKushnir) (Roman Kushnir)
* [amaierhofer](https://github.com/chriscz/mailboxer/commits/master?author=amaierhofer) (Andreas Maierhofer)
* [tonydewan](https://github.com/chriscz/mailboxer/commits/master?author=tonydewan) (Tony Dewan)
* [plentz](https://github.com/chriscz/mailboxer/commits/master?author=plentz) (Diego Plentz)
* [laserlemon](https://github.com/chriscz/mailboxer/commits/master?author=laserlemon) (Steve Richert)
* [daveworth](https://github.com/chriscz/mailboxer/commits/master?author=daveworth) (Dave Worth)
* [rafaelgg](https://github.com/chriscz/mailboxer/commits/master?author=rafaelgg) (Rafael Garcia)
* [joshblour](https://github.com/chriscz/mailboxer/commits/master?author=joshblour) (joshblour)
* [iamdeuterium](https://github.com/chriscz/mailboxer/commits/master?author=iamdeuterium) (iamdeuterium)
* [daveworth](https://github.com/chriscz/mailboxer/commits/master?author=daveworth) (Dave Worth)
* [parndt](https://github.com/chriscz/mailboxer/commits/master?author=parndt) (Philip Arndt)
* [atd](https://github.com/chriscz/mailboxer/commits/master?author=atd) (Antonio Tapiador)
* [mobilutz](https://github.com/chriscz/mailboxer/commits/master?author=mobilutz) (Lutz)
* [bennick](https://github.com/chriscz/mailboxer/commits/master?author=bennick) (Ryan Bennick)
* [rjst](https://github.com/chriscz/mailboxer/commits/master?author=rjst) (Ricardo Trindade)
* [fabianoalmeida](https://github.com/chriscz/mailboxer/commits/master?author=fabianoalmeida) (Fabiano Almeida)
* [supremebeing7](https://github.com/chriscz/mailboxer/commits/master?author=supremebeing7) (Mark J. Lehman)
* [searsaw](https://github.com/chriscz/mailboxer/commits/master?author=searsaw) (Alex Sears)
* [joshblour](https://github.com/chriscz/mailboxer/commits/master?author=joshblour) (Yonah Forst)
* [nhinze](https://github.com/chriscz/mailboxer/commits/master?author=nhinze) (Nhinze)
* [rockdog](https://github.com/chriscz/mailboxer/commits/master?author=rockdog) (Sven Steinheißer)
* [MarkFChavez](https://github.com/chriscz/mailboxer/commits/master?author=MarkFChavez) (Mark Chavez)
* [booleanbetrayal](https://github.com/chriscz/mailboxer/commits/master?author=booleanbetrayal) (Brent Dearth)
* [fastengineer](https://github.com/chriscz/mailboxer/commits/master?author=fastengineer) (Mehmet Aydogdu)
* [afamyial](https://github.com/chriscz/mailboxer/commits/master?author=afamyial) (Ankit Famyial)
* [KaoruDev](https://github.com/chriscz/mailboxer/commits/master?author=KaoruDev) (Kaoru Kohashigawa)
* [jerefrer](https://github.com/chriscz/mailboxer/commits/master?author=jerefrer) (Jérémy FRERE)
* [kevinjbayer](https://github.com/chriscz/mailboxer/commits/master?author=kevinjbayer) (Kevin Bayer)

---
layout: post
title:  "Message Retry as an alternative to Transactional Outbox"
date:   2021-06-22 21:03:36 +0530
categories: Microservices Outbox
---

[Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html) is a well known pattern in microservices, which helps us to ensure that both database write operation and publishing messages to a message broker occur atomically, that is, both or none.

If it's a first time you heard about the pattern, please google around. In this article i'll also assume that you know about [Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html) Pattern

Doing outbox is a lot of work in order to just send a message. Message(s) need to be stored in database, then a separate process should read and dispatches to message broker.

We can achieve the same thing with less effort, or can we ?

Hopefully you're not connecting to a message broker directly with bare SDK and sensibly using some predbuilt framework of sorts, which gives you an abstraction over the message broker, then you most likely have the Retry capability provided. That is, if the message processing fails, the message will be retried.

With retry in place, we can commit a transaction and then start dispatching messages, if failure happens at this stage, message will be retried and we can re-dispatch messages then.

The pseudo code would look something like this

1. Check if incoming message is already processed (Idempotency check)
2. If not, process the incoming message and dispatch outgoing messages
3. If yes, then skip message processing, restore outgoing messages and re-dispatch.

Imagine entity `User` with two attributes `Email` and `Balance`

Scenario 1 :


```
UpdateUserEmail : 
{
    Userid : Unique Identifier Of User,
    NewEmail : New Email Address
}
```

```
UserEmailUpdated : {
    Userid : Unique Identifier Of User,
    Email : Updated Email Address
}
```

1. The message `UpdateUserEmail` is received
2. Read the `User`, update it's email and store it back to the database.
3. Create `UserEmailUpdated` event and set `UserId` and `Email`
4. Application crashed and `UserEmailUpdated` can't be delieverd to message broker.
5. `UpdateUserEmail` will be retried, with idempotency mechanism in place, you know that message is already processed, so, you'll skip updating an user and only re-dispatch `UserEmailUpdated` event 

In Scenario 1, we can "restore" `UserEmailUpdated` event during the retry, because this event contains only `Email` and `UserId` attributes which can simply be copied from `UpdateUserEmail` command.

Though, make no mistake, there are scenarios when you can not simply restore an event.

Scenario 2 :

Say you have the following business logic : If the user has `gmail` address and his balance is `123` then debit him/her `100` unit. Whenever an unit is debited to user, `UnitDebittedToUser` event should be published.

Lets assume that User's current balance is `123`

1. `UpdateUserEmail` message is received, with `Email` equal to `usr@gmail.com`. User's balance became 223 and database transaction commited.
2. The application crashed.
3. Before the `UpdateUserEmail` is retried, the second message `UpdateUserEmail` is received with `Email` equal to `usr@yahoo.com`
4. Second `UpdateUserEmail` is processed, User Updated, Transaction commited and `UserEmailUpdated` event published.
5. Now first `UpdateUserEmail` command is retried, the message is already processed hence no need to update an user, although events need to be re-dispatched. Problem is that, there is no way we can restore `UnitDebittedToUser` event. User's balance is now `223`, hence logic like `If balance == 123 and emailDomain == 'gmail'` will evaluate negative.

Conlusion

Retry might at first seem like an alternative solution to Outbox, but the big advantage of Outbox is that it sets all the outgoing message in stone (That is, the message are stored in a database) hence no need to "restore" them during retry. As we saw, the problem with trying to retry an outgoing message in retry is that, after initial processing and before retry, underlying data in database might change in a way that you'll either restore invalid outgoing message or lose it alltogether.
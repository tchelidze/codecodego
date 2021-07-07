---
layout: post
title:  "Message Retry as an alternative to Transactional Outbox"
date:   2021-06-22 21:03:36 +0530
categories: Microservices Outbox
---

[Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html) is a well known pattern in microservices, which helps us to ensure that both database write operation and publishing messages to a message broker occur atomically, that is, both or none.

If it's the first time you heard about the pattern, please google around. In this article i'll also assume that you know about [Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html) Pattern

Doing outbox is a lot of work in order to just send a message. Message(s) need to be stored in database, then a separate process (Message Relay) should read and dispatch message(s) to a message broker.

We can achieve the same thing with much less effort, or can we ?

Hopefully you're not connecting to a message broker directly with a bare SDK and sensibly are using some predbuilt framework of sorts ([NServiceBus](https://particular.net/nservicebus), [Eventuate](https://eventuate.io/), etc), which gives you an abstraction over the message broker, then you most likely have the Retry capability provided. That is, if the message processing fails, the message will be retried.

With retry in place, we can commit a transaction and then start dispatching outgoing messages, if a failure happens at this stage, incoming message will be retried and we can re-dispatch outgoing messages during a retry.

The pseudo code would look something like this

1. Check if incoming message is already processed (Idempotency check)
2. If not, process the incoming message and dispatch outgoing messages
3. If yes, then skip message processing, restore outgoing messages and re-dispatch.

Imagine entity `User` with two attributes `Email` and `Balance`

Scenario 1 :

Given the following Command (Incoming message) and an Event (Outgoing message)

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
2. Read the `User`, update it's email and store it back to a database.
3. Create `UserEmailUpdated` event and set `UserId` and `Email`
4. Application crashed and `UserEmailUpdated` can't be delieverd to message broker.
5. `UpdateUserEmail` will be retried, with idempotency mechanism in place, you know that message is already processed, so, you'll skip updating an user and only re-dispatch `UserEmailUpdated` event 

In Scenario 1, we can "restore" `UserEmailUpdated` event during the retry, because this event contains only `Email` and `UserId` attributes which can simply be copied from `UpdateUserEmail` command.

Though, make no mistake, there are some scenarios when you can not simply restore an event.

Scenario 2 :

Say you have the following business logic : If the user has `gmail` address and his balance is `100` then he debit him/her `50` unit. Whenever an unit is debited to user, `UnitDebittedToUser` event should be published.

Given the following Command and an Event

```
CreditUserBalance :
{
    Userid : Unique Identifier Of User,
    Amount : The amount to credit user with
}
```

```
UnitDebittedToUser : {
    Userid : Unique Identifier Of User,
    Amount : The amount debitted to user
}
```

User's current balance is `100`

1. `UpdateUserEmail` message is received, with `Email` equal to `usr@gmail.com`.  User's balance is set to 150 and user is stored back to a database.
2. The application crashed.
3. Before the `UpdateUserEmail` is retried, the concurrent message `CreditUserBalance` is received with `Amount` equal to `20`
4. `CreditUserBalance` is processed, User's balance is set to 170 and user is stored back into a database.
5. `UpdateUserEmail` is Retried, the message is already processed hence no need to update an user, although events need to be re-dispatched.
6. User's balance is now `170` and there is no (unless super complex) way we can  restore `UnitDebittedToUser` event. As a result, `UnitDebittedToUser` is LOST.

Conlusion

Retry might at first seem like an easy alternative to the Outbox, but the big advantage of Outbox is that, it sets all the outgoing message in stone (That is, the message are stored in a database) hence no need to "restore" them during the retry.

 As we saw, the problem with trying to re-dispatch an outgoing message during a retry is that, after initial processing and before a retry, underlying data in a database might change in a way that you'll either restore invalid outgoing message or lose it alltogether.
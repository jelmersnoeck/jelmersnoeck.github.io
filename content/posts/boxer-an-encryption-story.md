---
title: "Boxer: an encryption story"
date: 2018-06-28
---

This article was originally published on [the Manifold blog](https://www.manifold.co/blog/boxer-an-encryption-story-fadcc9099ba6)

![header](/images/boxer-000.png)

At Manifold, we take security seriously. We store sensitive information, credentials, so we've architected Manifold from the ground up with an encryption system in place. We called this system Boxer.

In this blog post we'll go over some of the details on how we've implemented this. By doing so, we hope we can clarify our data safety vision, and introduce some security patterns for you to consider in your own designs.

## TL;DR: the why and how

At the start of the Boxer project, we wanted to ensure that only the credential's owners could decrypt their own data. To achieve this, we need to have a separate encryption key per owner.

We encrypt the owner specific key with a region master key, which we then store in our database. By doing this, we can also replicate our data across regions — and cloud providers — with ease. We only have to re-encrypt the owner's keys; the data remains the same.

![2 separate database layers, each with their own encryption key. A master key per region.](/images/boxer-001.png)
*2 separate database layers, each with their own encryption key. A master key per region.*

## Encryption Layers

As mentioned before, we have multiple layers of encryption. In this section we'll take a deep dive into each of these layers.

**Data Layer**

At the core, we need to encrypt data with a given key. To do this, we use [SecretBox](https://godoc.org/golang.org/x/crypto/nacl/secretbox). SecretBox is a symmetric-key algorithm, which allows us to use the same key for encryption and decryption. This means that we can use our generated master key to encrypt our data to store it, but also decrypt it when reading it.

The Data Layer is where we actually store the encrypted credentials. We can now replicate this layer across different regions or cloud providers. Due to our separate Principal Layer, we don't have to worry about re-encrypting the data.

We use this algorithm — albeit abstracted — for all the sensitive data we want to encrypt. SecretBox is optimized for small messages, which makes it ideal for our messages: very short strings.

**Principal Layer**

Once we have our data securely stored, we also need to think about storing the master key securely. As we mentioned before, we wanted to make sure that no one could decrypt anyone else's secrets in case of a breach.

To do this, we've created a Principal Layer. This layer looks at who the owner of a credential is, either a team or user. Based on this owner, we then create a new random token which we use to encrypt the credentials in the Data Layer. We then use [RBAC](https://en.wikipedia.org/wiki/Role-based_access_control) to determine if a user has access to this resource or not. You need to be a member of a team *and* have the right permissions in order to read credentials.

By doing this, we ensure that in case of [SQLi](https://en.wikipedia.org/wiki/SQL_injection) a user can't get any other user's information out.

**Cloud Layer**

Our top layer is our Cloud Layer. This is a service each cloud provider provides. In our case, this is [AWS KMS](https://aws.amazon.com/kms/) but it could be any Key Management Service.

We use this encryption layer as the master key to our Principal Layer. This ensures that when there is a database breach (through SQLi for example), intruders don't have access to our user data. They would still need access to our KMS keys.

KMS keys are usually limited to a specific region. This means we'd have to re-encrypt our data per region. By separating the Data Layer from the regional master key — KMS, we can leave the data as is. Now we only have to re-encrypt the Principal Layer keys, and we can use the decrypted version to read our data.

## Hooking it all up

To do everything described earlier, we made an abstraction layer on top of our database. First of all, this makes it easier to do all the encrypting and decrypting. But more important, this ensures that we actually encrypt our data, and that we do it correctly.

![When a request comes in, it gets verified with RBAC. Boxer then takes care of determining the correct OwnerID and decrypting the data accordingly.](/images/boxer-002.png)
*When a request comes in, it gets verified with RBAC. Boxer then takes care of determining the correct OwnerID and decrypting the data accordingly.*

**Interfaces**

At Manifold, we're big on using Go. In a [previous post](https://blog.manifold.co/how-go-interfaces-can-facilitate-switching-external-services-619cc478e20a), Luiz explains how you can use interfaces to switch between services. We heavily rely on Go Interfaces for abstracting our encryption. This enables our developers to use this setup with ease and always ensures the data gets encrypted.

We have a Querier interface which defines methods like `Get`, `Create`, `Update` and `QueryOne`. Our default implementation has a simple mapping to our Database engine (PostgreSQL). For Boxer, we've implemented a layer on top of this implementation. This automatically takes care of encrypting and decrypting. This makes it accessible to our developers and ensures data safety.

For security reasons, when you want to use this Querier, you'll have to specify who the owner is that you're querying for. We base this owner off the authenticated user. As mentioned before, this owner can only decrypt their own data since each owner gets their own key. This means that there is no option to decrypt someone else's credentials.

## Wrapping up

Due to the sensitive nature of our data, we had to think about security from the start. We've implemented a 3 layer encryption setup which gives us several benefits. It protects users from SQL injections and data breaches. It also helps us scale our platform and allows us to serve data across regions. If you encounter similar design needs, we encourage you to consider the pattern.

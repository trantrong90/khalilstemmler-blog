---
templateKey: article
title: "Command Query Segregation | Object-Oriented Design Principles w/ TypeScript"
date: '2019-08-29T10:04:10-05:00'
updated: '2019-08-29T10:04:10-05:00'
description: >-
  CQS (Command-Query Segregation) is a design principle that states that a method is either a COMMAND that performs an action OR a QUERY that returns data to the caller, but never both.
tags:
  - Command Query Segregation
  - Object-Oriented Programming
category: Software Design
image: /img/blog/design-principles/object-oriented-design-principles.png
published: true
---

I recently discovered a programming term, <u>Heisen_bug_</u>. 

No, I'm not talking about about a certain chemistry teacher turned criminal drug-lord. 

**HeisenBUG**.

It's a pun on the name of a certain German physicist, <a target="_blank" href="https://en.wikipedia.org/wiki/Werner_Heisenberg">Werner Karl _Heisenberg_</a>.

Heisenberg's discovery in particle physics was the <u>uncertainty principle</u>. It stated that "the very act of observing [a particle] alters the position of the particle being observed, making it impossible (in theory) to accurately predict its behavior" [^1].

> Is it just me or does that remind you of your worst debugging nightmares?

**HeisenBUG** takes after the uncertainty principle, referring to scenarios where it's incredibly hard to find a bug. It specifically refers to scenarios where the bug seems to disappear or act differently when we make attempts to take a closer look at it.

In programming, unexpected **side-effects** are largely the cause of such bugs.

> If asking a question _changes the answer_, we're likely to run into problems.

--- 

## Command-Query Segregation (CQS)

Command-Query Segregation (CQS) is a design principle (while not strictly object-oriented) that states that:

> a _method_ is either a `COMMAND` that performs an action OR a `QUERY` that returns data to the caller, but never both.

In simpler terms, "asking a question shouldn't change the answer".

### _Why does this matter?_

The idea is to **separate the code paths** for operations that _change the system_ from those that simply _request_ data from the system.

By enforcing this segregation, the code becomes simpler to understand. _Is this changing something, or just fetching something_? When a method does _both_ (**changes** the state of the application _and_ **retrieves** data), it becomes a lot harder to understand it's **true purpose**. 

This can lead to <u>really hard to reason about</u> application state.

---

## A messy Job Recommendations API Design

Imagine we were building a **Job Recommendations Service**.

Assume we have two API routes:

```
GET /jobs         - Returns jobs for me to view
GET /jobs/:jobId  - Returns a particular job
```

Consider if everytime I did a `GET` to `/jobs/:jobId`, it **changed the state of the system** by changing what comes back when I do `GET` to `/jobs`.

Eventually, as I view more jobs with `/jobs/:jobId`, I'll see more relevant jobs in my calls to `/jobs`.

In theory, that's one way to build out job recommendations in our system.

But this type of design makes the `/jobs` API _incredibly inconsistent_. 

Consider how hard it would be to test and validate it's working properly.

Also, consider the actors in this system are as follows:

- `JobSeekers`: Job seekers are people who are actually looking for jobs
- `Recruiters`: Recruiters work for companies and try to get job seekers to apply to jobs
- `Employers`: Employers are the people who post the jobs
- `Public`: Anonymous users can also view jobs on the job board without an account

Assume that every actor was able to use the `/jobs/:jobId` API call to retrieve a posting.

Should the system change for every actor that fetches a job posting with `/jobs/:jobId`?

Of course not, `JobSeekers` are probably the only group that this should apply against. 

This **feels really complex** because several user groups are reliant on the same resource, but we're trying to apply side-effects for **one of them in particular**.

You can be sure that using a design like this, there would be a dirty `if` statement in there somewhere:

<div class="filename">getJobByJobId.ts</div>

```typescript
interface Request {
  userId?: string;
  jobId: JobId;
}

class GetJobByJobIdUseCase implements UseCase<Request, Promise<Job>> {
  ...
  execute (request: Request): Promise<Job> {
    const { userId, jobId } = request;
    
    if (!!userId) {
      const user = await this.userRepo.findById(userId);

      const isJobSeeker = !!user.roles.find((r) => r === 'JobSeeker');
      if (isJobSeeker) {
        // dirty, log job view
      }
    }

    const job = await this.jobRepo.getJobByJobId(jobId);
    ...
  }
}
```

### A better design

Let's be explicit about the `COMMANDS`, `QUERIES`, what changes the system, and what retrieves data.

We can improve the design by **extracting** the side-effect into a `COMMAND`-like POST call to **log the job view** separately from retrieving it.

```
GET /jobs                 - Returns all jobs 
GET /jobs/:jobId          - Returns a particular job
POST /jobs/:jobId/view    - 🔥 Logs a job view 
```

This tremendously simplifies the code paths for both **logging a job view** and **retrieving a particular job**.

We can improve the design even further by extracting the functionality to see recommended jobs from within it's own `/jobs/recommendations` API.

```
GET /jobs                     - Returns all jobs 
GET /jobs/:jobId              - Returns a particular job
POST /jobs/:jobId/view        - Logs a job view
GET /jobs/recommendations     - 🔥Returns my personal job recommendations 
                                  (used by JobSeekers only)
```

Usage of this API from the UI would mean that everytime we perform a `GetJobById` `QUERY`, we accompany that with a `LogJobView` `COMMAND`.

This way, we have two **separate code paths**: one for _changing the system_ and for _pulling data out of the system_. We can rest safely knowing that if we change anything with regards to `QUERY`-ing, it won't break anything in regards to how we execute `COMMAND`s, and vice-versa.

## Violation of the principle at the code level

Consider you wrote the following `postComment` method in a **Comment Moderation System**. 

<div class="filename">commentRepo.ts</div>

```typescript
interface Comment {
  name: string;
  url: string;
  content: string;
  id: string;
}

class CommentRepo {
  ...
  postComment (name: string, url: string, content: string): Promise<Comment> {
    const CommentSequelizeModel = this.models.Comment;

    // Post comment
    const comment = await CommentSequelizeModel.create({ name, url, content });
    return CommentMap.toDomain(comment); // with commentId on it
  }
}
```

First of all, take a look at method signature:

`postComment (name: string, url: string, content: string): Promise<Comment>`

The name implies that the operation will be a `COMMAND`, but it returns a value as well, violating the principle.

Perhaps returning the `Comment` created can be justified. Assume a `CommentService` exists, and in order to craft a <a target="_blank" href="https://slack.com/intl/en-ca/">Slack</a> channel message notifying us of a new comment, we needed the `commentId` of the newly created  `Comment` returned from the `CommentRepo`'s `postComment` method.

<div class="filename">commentService.ts</div>

```typescript
class CommentService {
  ...
  async postCommentAndPostToSlack (name: string, url: string, content: string) {
    const comment = await this.commentRepo.postComment(name, url, content);

    // Needs comment.commentId in order to craft the message.

    this.slackService.sendMessage(`
      New comment posted:
        => Name: ${name}
        => Url: ${url}/commentId/${comment.id}
        => Content: ${content}
    `)
  }
}
```

So then...

### What's wrong with this code?

- While the `CommentRepo` method is deceptively named `postComment`, it's not only responsible for posting the comment, but also for retrieving the comment that was posted. Developers reading the method signature might get confused as to the **single responsibility** of this method. `QUERY` capability should be delegated to a new method, perhaps `getComment(commentId: string)`.
- There's an issue in not generating the id for the `Comment` from within the [domain layer](/articles/enterprise-typescript-nodejs/clean-nodejs-architecture/), but leaving it up to the persistence layer (Sequelize) as shown here. That can lead to blatant violation of the principle _in order to know_ the identifier of the [Entity](/articles/typescript-domain-driven-design/entities/) just saved to the database. That poor design forces calling code executing a `COMMAND` to not _only_ know if the `COMMAND` succeeded or failed, but also forces the `COMMAND` to return the value if successful.

### Fixing it

In addition to switching to creating the entire `Comment` domain model from within the Domain Layer (using [Value Objects](/articles/typescript-value-object/) and  [Entities](/articles/typescript-domain-driven-design/entities/) containing the [UUID identifier](/articles/auto-increment-or-uuid/)), we can segregate the `COMMAND` from the `QUERY` aspect of the `postComment` method by introducing a new method, `getComment`.

<div class="filename">commentRepo.ts</div>

```typescript
class CommentRepo {
  ...
  postComment (comment: Comment): Promise<void> {
    const CommentSequelizeModel = this.models.Comment;
    // Post comment
    await CommentSequelizeModel.create(comment);
    return;
  }

  getComment (commentId: CommentId): Promse<Comment> {
    const CommentSequelizeModel = this.models.Comment;
    const createQuery = this.createQuery();
    createQuery.where['comment_id'] = commentId.id.tostring();
    const comment = await CommentSequelizeModel.findOne();
    return CommentMap.toDomain(comment)
  }
}
```

And now, from `CommentService`, we should be able to send the Slack message without relying on the return value from a `COMMAND`.

<div class="filename">commentService.ts</div>

```typescript
class CommentService {
  ...
  async postCommentAndPostToSlack (name: string, url: string, content: string) {
    const comment: Comment = Comment.create(name, url, content);
    await this.commentRepo.postComment(comment);

    // Needs comment.commentId in order to craft the message.

    this.slackService.sendMessage(`
      New comment posted:
        => Name: ${name}
        => Url: ${url}/commentId/${comment.id}
        => Content: ${content}
    `)
  }
}
```

## Quiz

Let's see how well I explained that.

**Which of these are valid `COMMAND`s?**

1. `getComment (commentId: CommentId): Promise<void>`
2. `createJob (job: Job): Promise<Job>`
3. `postComment (comment: Comment): Promise<Comment[]>`
4. `approveComment (commentId: CommentId): Promise<void>`

<div class="flex flex-centered align-center">
  <button 
    style="min-width: 165px;"
    onclick="showQuizAnswer('quiz-1')" 
    class="submit-button purple">Show answers
  </button>
  <div 
    id="quiz-1" 
    style="margin-left: 0.5rem; display: none;">Only #4.
  </div>
</div>

<br/>

---

Which of these are valid `QUERIES`?

1. `getAllVinyl (): Promise<Vinyl[]>`
2. `getVinylById (vinylId: VinylId): Promise<Vinyl[]>`
3. `getAllVinyl (): Promise<void>`
4. `getAllVinylById (vinylId: VinylId): Promise<Vinyl>`

<div class="flex flex-centered align-center">
  <button 
    style="min-width: 165px;"
    onclick="showQuizAnswer('quiz-2')" 
    class="submit-button purple">Show answers
  </button>
  <div 
    id="quiz-2" 
    style="margin-left: 0.5rem; display: none;">#1, #2, and #4, although #2 is confusing to return an array of vinyl in regards to one vinylId.
  </div>
</div>

<br/>

---

## CQS is a constantly occuring principle in several contexts of software development

### CQS in CRUD

**C**reate **R**ead **U**pdate and **D**elete (CRUD) is often how we think about designing trivial MVC applications. Each operation in CRUD perfectly fits the definition of either a `COMMAND` or a `QUERY`.

- CRUD Commands: `Create`, `Update`, `Delete`
- CRUD Queries: `READ`

### CQS in RESTful HTTP

CQS also syncs up with the principles of RESTful HTTP. An HTTP method is also either a `COMMAND` or a `QUERY`.

- HTTP Commands: `POST`, `PUT`, `DELETE`, `PATCH`
- HTTP Queries: `GET`

In times where it may be challenging to think about the behaviour of a particular RESTful API route, think back to the CQS principle.

- Create User: POST - `/api/users/new`
- Get a user: GET - `/api/users/:userId`

### CQS in SQL

Nearly every operation we do in SQL is a `COMMAND` and only one operation is a `QUERY`. I'm sure you can guess which one (hint: it rhymes with "REFLECT").

### CQS in Use-Case Design

[Projects beyond simple MVC](/articles/enterprise-typescript-nodejs/when-crud-mvc-isnt-enough/) are centered around the [use cases](/articles/enterprise-typescript-nodejs/application-layer-use-cases/) of the application. These are the **features** for an individual **group** of people/`Actors`.

Every use case is _strictly_ a command or a query.

In a `Blog` subdomain,

- Commands: `CreatePost`, `UpdatePost`, `DeletePost`, `PostComment`, `UpdateComment`
- Queries: `GetPostById`, `GetAllPosts`, `GetCommentById`, `GetAllCommentsForPost`

---

## CQS in Domain-Driven Design Architecture with CQRS

In Domain-Driven Design, it makes sense to separate the `READ` models from the `WRITE` models. `WRITE`s usually take a little bit more time to finish executing because a `WRITE` to [an aggregate](/articles/typescript-domain-driven-design/aggregate-design-persistence/) requires the aggregate to be fully-constituted and contain everything within it's consistency boundary in order to enforce [invariants](/wiki/invariant/).

Using the same `READ` model as a `WRITE` model can be **expensive** as we've explored in [this article](/articles/typescript-domain-driven-design/one-to-many-performance/).

It makes more sense to enable the `READ` models to be used for all `QUERIES` and the `WRITE` model to be used for the `COMMAND`s.

This separation into **two different types of models** is an architectural pattern called **Command Query _Responsibility_ Segregation** (CQRS).

If you're doing [Domain-Driven Design](/articles/domain-driven-design-intro/), it will likely be hard for you to avoid it.

### Performing Reads in CQRS

![Command Query Responsibility Segregation | Reads](/img/blog/design-principles/cqrs-reads.svg)

### Performing Writes in CQRS

![Command Query Responsibility Segregation | Writes](/img/blog/design-principles/cqrs-writes.svg)

## Summary

- In theory, the vast majority of operations that we write in programming only either do one or the other (change the application state, or retrieve some data), but there  are times where it can be unclear.
- Being conscious of this principle can clear up any ambiguity towards if a method of a `COMMAND` or a `QUERY` and if we expect any side effects to change the state of the app. This reduces bugs, improves readability, and makes code more testable.
- CQS usually isn't thought about in simple CRUD applications because each of the operations in **C**reate **R**ead **U**pdate and **D**elete operations are obviously either a command or a query.
- CQS is well-suited towards complex domains where Domain-Driven Design is most useful.
- There exists another technique called CQRS which is essentially CQS, but at the architectural level. 
- In DDD apps, use cases can sometimes ambiguously feel like a `COMMAND` and a `QUERY`, like the `GetJobById` `QUERY` (_Job board example_). It's important, for performance, to be aware of what really is a `COMMAND` and what really is a `QUERY`, and use CQRS to separate the read and write models.


[^1]: [Uncertainty Principle](https://en.wikipedia.org/wiki/Uncertainty_principle) - Fascinating wikipedia entry on the principle also sometimes known as _The Heisenberg Effect_.
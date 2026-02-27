- Feature Name: `crates_io_username_identity`
- Start Date: YYYY-MM-DD
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Someday, we would like to enable people to log in to crates.io with other services in addition to
GitHub. This RFC is not yet about adding other services for login. It is proposing that crates.io
change to have the concept of a "crates.io username" separate and possibly different from users'
GitHub usernames. Crates.io needs this change to make authenticating with different services
possible while minimizing confusion.

> 🚨 After this RFC is accepted and implemented, you will still only be able to log in to crates.io
> via GitHub. This is a prerequisite of the eventual goal to add other methods of logging in. 🚨

# Motivation
[motivation]: #motivation

Crates.io's code currently has a one-to-one mapping between crates.io accounts and GitHub accounts.
The URL `https://crates.io/users/some_username` displays the crates owned by the user with the
GitHub account `some_username`, and running `cargo owner add some_username` adds `some_username` as
an owner of the current crate. Owners of a crate appear in the sidebar. Crate ownership conveys
trust.

Eventually (after future RFCs and additional work after this RFC), we'd like to add the ability to
create crates.io accounts by logging in via OAuth with accounts from services other than GitHub, as
well as associating OAuth accounts from multiple services to one crates.io account.

The same username on GitHub is not guaranteed to belong to the same person on other services, and
one person's usernames across different services are not guaranteed to be the same. When we add
more services, crates.io's codebase needs to be able to handle these situations and clearly convey
crates.io user identities to minimize the possibility of confusion or deliberate impersonation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Today, crates.io usernames always match the GitHub username of the account used to log in to
crates.io (with exceptions for renamed or deleted GitHub accounts that will be discussed below).

After this change, there will be the concept of a crates.io username that may or may not match the
GitHub username of the associated account. When there are multiple ways of logging in in the
future, the crates.io username may or may not match the usernames on the other services. All
existing active (that is, not deleted or renamed) accounts will have their crates.io username set
to their current username, their GitHub username.

When you visit your account settings page, you will be able to edit your username to anything that
isn't already claimed as a crates.io username (We could choose to wait to allow username editing
until we have multiple ways of logging in, but we could also choose to enable username editing
sooner). Thus, crates.io usernames will become first-come-first-served as crate names are today.
Crates.io admins will not change an account's username without the consent of the current username
holder (see [Unresolved Questions](#unresolved-questions) about username squatting).

When you visit a user's page at `https://crates.io/users/example_username`, see a user account
listed as an owner of a crate in the crate's sidebar, or run `cargo owner add example_username`
and the account's crates.io username differs from the GitHub username associated with the account,
you will see a warning icon similar to ⚠️ and text that says something like "username does not match
GitHub username". Given that the common case, and what people are used to being able to know, will
be that the GitHub and crates.io usernames will match, this will make it obvious in cases where
that assumption does not hold.

When you create an account on crates.io with an OAuth account (GitHub or otherwise), whether or not
the associated OAuth account's username is currently claimed on crates.io, you will be asked to
register your crates.io account by choosing a username that hasn't yet been taken on crates.io. The
crates.io username field will be prefilled with the associated OAuth account's username and an
indication of whether that username is available on crates.io or not.

## Renamed and deleted accounts

GitHub allows users to change their username (but keep the same GitHub ID number so that crates.io
can know it's the same account) or delete their account, which makes the username available for
someone else to claim (with a different GitHub ID number than was previously associated with it).

Crates.io currently does not proactively check GitHub for account status. If someone takes these
actions:

1. Creates GitHub account with the username "example"
2. Logs in to crates.io with that account so that they have the crates.io username "example"
3. Renames their GitHub account to "something_else"
4. Never logs in to crates.io with that GitHub account again

Their crates.io username will remain "example", until such a point that they decide to log in to
crates.io again. Currently, crates.io will then update their username in our database to match.

This RFC proposes decoupling GitHub account renaming from crates.io username completely, so that
GitHub account renames do NOT automatically become crates.io account renames.

A similar situation occurs when a user deletes their GitHub account. The username "example" will
then be available for someone else to claim on GitHub, but will remain claimed on crates.io.

If a different user does one of the following:

- Creates the GitHub account "example" and logs in to crates.io
- Tries to edit their username to "example"
- Logs in to crates.io with an account on some service other than GitHub with username "example"

At that point, crates.io will:

- See that the crates.io username "example" is taken
- Require the user with the GitHub username "example" to pick a different crates.io username

If the old "example" account had it via their associated GitHub account (and thus didn't have the
mismatch ⚠️ warning discussed above), then a new associated GitHub account logs in with the GitHub
username "example" (and a different GitHub ID), at that point we know the GitHub account "example"
does NOT belong to the crates.io account "example" and the crates.io account "example" should get
the mismatch ⚠️ warning. TODO should we just proactively monitor GitHub account renames rather than
doing this check on OAuth login?

If a user manually changes their crates.io username to `best_rust_programmer_ever` (and doesn't
have the matching GitHub account and thus has the warning symbol), and then later someone creates a
GitHub account with the username `best_rust_programmer_ever` and logs in to crates.io, the GitHub
user `best_rust_programmer_ever` will need to choose a different crates.io username. Both crates.io
accounts will have the warning symbol. The latter user may see this as unfair, but this is where
the first-come-first-serve policy should be enforced.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

- The `users.gh_login` column is currently not unique because of the rename/delete scenarios above.
  Write a script that goes through all duplicate `gh_login` values and queries the GitHub API for
  the associated GitHub ID's current status and either update the `gh_login` value to the current
  GitHub name (for renames) or update the `gh_login` value to a unique value
  (`oldname_archived_[randomdigits]`) (for deletions).
- Once the `users.gh_login` column's values are unique, add a unique constraint in the database.
- Rename `users.gh_login` column to `users.username` (or `users.login`, but this wouldn't be used
  to log in so I would go with `username`. And not `users.name` despite that being less repetitive
  as that field is currently used for the "display name" concept, unless we first rename
  `users.name` to `users.display_name` or remove that field entirely as discussed in [Unresolved
  Questions][#unresolved-questions])
- When we get a GitHub OAuth response for someone signing up or signing in, make the following
  changes:
  - Rather than looking up the GitHub ID in the `users` table, look it up in the `oauth_github`
    table to make the `oauth_github` table the source of truth about GitHub accounts rather than
    the `users` table.
  - If an `oauth_github` record exists with the provided GitHub ID:
    - Update the username, token, and avatar on the `oauth_github` record to what was specified
      from GitHub.
    - Do not update the crates.io username on the `users` record associated with the `oauth_github`
      record, even if the GitHub username has changed.
  - If an `oauth_github` record doesn't exist with the provided GitHub ID (and thus a `users`
    record doesn't exist either):
    - Insert a new `users` record with the crates.io username the user provided during signup
    - Then insert an associated `oauth_github` record with the provided GitHub username, token, and
      avatar
  - If at any time in these operations, we get a violation of the `users.username` uniqueness
    constraint because the crates.io username is already taken, ask the user for a different
    username until they pick a username that isn't taken. Carry the other information along in the
    session.
- When we get a request for `https://crates.io/users/example`, do the query `SELECT * FROM users
  WHERE users.username = 'example';`. Also query for associated `oauth_github` records (and
  eventually all other `oauth_*` associated tables) to be able to display links to the associated
  GitHub account.
- When we get a request through `cargo owner add example`, also do the query `SELECT * FROM users
  WHERE users.username = 'example';`. Also query for associated `oauth_github` records (and
  eventually all other `oauth_*` associated tables) to be able to return a link to the associated
  GitHub account in the response provided to Cargo to display for the user adding the owner to use
  to verify they have just invited the intended person (and remove the owner if not).

# Drawbacks
[drawbacks]: #drawbacks

- Impedes signup flow if you have to choose a username or try multiple usernames before finding an
  available one
- Could cause confusion during signup and user lookup
- People who can't or don't want to have their GitHub username and crates.io username match will
  have a warning by their username that they might not want to have there and might imply there's
  something wrong or untrustworthy about their account when that isn't the case

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

We could choose to diverge from crates.io's current behavior more than proposed here, such as:

- We could force everyone to specify whether they mean the crates.io username or GitHub username
  for every lookup, to force education that they're no longer guaranteed to be the same. That is,
  if someone runs `cargo owner add example`, we'd return an error and ask them to rerun `cargo
  owner add cratesio:example` or `cargo owner add github:example` explicitly. This could be
  confusing for the most common case where these refer to the same user, but would be a way to
  force communication with people that something is changing.

## "Disambiguation page" alternative

We could choose not to have a `username` field on the `users` table at all. When visiting
`https://crates.io/users/example`, crates.io would always look up usernames in all available
`oauth_*` tables. If only one `user` record was associated with `oauth_*` records that had the
username `example`, we'd display that user's information. This would nicely handle the most common
case of having one user with a unique GitHub username.

If, however, there is one crates.io `user` record (id 1234) associated with the `oauth_github`
record that has the username `example`, and another crates.io `user` record (id 5678) associated
with the `oauth_gitlab` record that has the username `example`, we could show content similar to a
Wikipedia "disambiguation page", something like:

> There are multiple users with the username "example". Did you mean:
>
> - [example on GitHub](https://crates.io/users/example/1234)
> - [example on GitLab]((https://crates.io/users/example/5678)

and then you'd have to click an extra time. We could support direct linking to these users either
by including their crates.io user record ID (something like
`https://crates.io/users/example/1234`), or by including the name of the service where they hold
the username (something like `https://crates.io/users/example/github`).

For the `cargo owner add` CLI, we could show similar disambiguation text and exit with an error:

```
$ cargo owner add example

ERROR: There are multiple users with the username "example".

If you meant https://github.com/example, rerun with `cargo owner add github:example`.
If you meant https://gitlab.com/example, rerun with `cargo owner add gitlab:example`.
```

The disambiguation page and extra specification for users who happen to have colliding usernames
with users from other services is a bit more friction and annoyance for those people, through no
real fault of their own. However, this case should be fairly rare.

Looking up a user would be more complex and require more queries to more tables, which may impact
overall performance.

This idea may not match people's expectations of how crates.io usernames work, causing confusion.
If a GitHub user with the username `best_rust_programmer_ever` was well known in Rust spaces but
actually never logged in to crates.io, but someone else claimed the username
`best_rust_programmer_ever` on GitLab and _did_ log in to crates.io, the content on
`https://crates.io/users/best_rust_programmer_ever` would only show the GitLab user's information
with no indication that the GitHub user even exists. We'd need to make the page content clear that
there was only a GitLab account attached, not a GitHub account attached as most people would expect
in most cases.

# Prior art
[prior-art]: #prior-art

Crates.io appears to be unique among the major OSS package registries in only offering GitHub
OAuth, so there aren't direct lessons we can draw from other ecosystems. Here are a few examples:

[PyPI](https://pypi.org) [does not currently support changing a
username](https://pypi.org/help/#username-change). Instead, you can create a new account with the
desired username, add the new account as a maintainer of all the projects your old account owns,
and then delete the old account, which will have the same effect. There is no OAuth support.

[npm](https://www.npmjs.com/) (JavaScript) does not have any OAuth login mechanisms. [Their
policies](https://docs.npmjs.com/policies/disputes) state they are "extremely unlikely to transfer
control of a username, as it is totally valid to be an npm user and never publish any packages".
[It is not currently possible to change your npm
username](https://docs.npmjs.com/changing-your-npm-username) other than creating a new account and
migrating data manually. When npm accounts are deleted, usernames become available for anyone to
claim again after 30 days.

[Maven Central](https://central.sonatype.com/) (Java) allows you to create an account and log in
via email or OAuth with Google, GitHub, or Microsoft. There is no way to rename, update or change
your Maven Central username. If you want a different username, you have to create a new account.
However, usernames don't appear to be as important as they are on crates.io. Maven Central is
organized around domain-based namespaces registered through DNS, and it's the namespace that
conveys authority.

[Keybase](https://keybase.io/) is a service that tries to make working with public key cryptography
easier. They have ways of proving ownership of various accounts on other services to help people
ensure they're communicating with the account that belongs to the intended person. Keybase also has
a CLI with a confirmation flow that we could use as inspiration for the `cargo owner add` user
flow. See [the Keybase documentation](https://book.keybase.io/docs/server), under the heading "Step
3: the human review":

> Recall, in Step 2 your client proved "maria" has a number of identities, and it cryptographically
> verified all of them. Now you can review the usernames it verified, to determine if it's the
> maria you wanted.
>
> ```
> ✔ maria2929 on twitter: https://twitter.com/2131231232133333...
> ✔ pasc4l_programmer on github: https://gist.github.com/pasc4...
> ✔ admin of mariah20.com via HTTPS: https://mariah20/keybase.tx...
>
> Is this the maria you wanted? [y/N]
> ```

With `cargo owner add`, once we support multiple logins, the CLI could look something like this:

```
$ cargo owner add carols10cents

Crates.io account `carols10cents` is associated with:
✔ https://github.com/carols10cents
✔ https://brand-new-code-hosting-platform.dev/some_other_username

Is this the `carols10cents` you wanted? [y/N]
```

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How would we define "squatting" of usernames that would be clear cases for admins to make
  available again? Accounts that don't have any crates and that don't have any valid associated
  OAuth logins (that is, all the associated accounts have been deleted)?
- We currently have the concept of a "display name" that is associated with and managed through
  your GitHub "display name" that's currently used on crate pages for owners and user pages. Do we
  want to have an authentication-independent "display name"? Or should we get rid of the "display
  name" concept completely and only show crates.io usernames everywhere? It seems like the display
  name would be a vector for possible impersonation (and I think that's technically possible today,
  but I'm not sure whether GitHub's policies would allow that)
- Avatars are also indicators of identities. Once we have multiple authentication services, each
  possibly providing an avatar, which do we display when a crates.io user account has different
  accounts associated and different avatars? Do we allow them to pick, and/or upload a completely
  unaffiliated crates.io avatar (which we'd then have to host; we currently don't host avatars)?
  Again, I think impersonation via avatar is technically possible today with only GitHub unless
  GitHub policy enforcement disallows that, and I don't think the decision on avatar resolution is
  as important as username resolution, but it might make implementation/database queries nicer if
  we make a similar decision with avatars as with usernames.
- We have a list of reserved crate names that no one may register that includes top-level Rust
  standard library modules and keywords, reserved Windows filenames, and some swear words or slurs
  (which will never be exhaustive but contains the most common ones in English). We'll probably
  need to have a similar list of reserved usernames that no one may use; GitHub's Terms of Service
  is providing us some protection currently that we'd need to manage ourselves.
- We are not yet committing to the support of organization/team owners from other services, but
  adding teams already requires specifying a literal `github:` before the `org:team` when adding
  team owners so there shouldn't be as much confusion around the identity of a team if we choose to
  add different team owners via different services.

# Future possibilities
[future-possibilities]: #future-possibilities

This functionality change would also enable a way of creating crates.io accounts without any
associated identity/reputation, only an email address. But this opens more potential for spam and
abuse as it's easier to create anonymous email addresses than it is to maintain accounts in good
standing on services like GitHub. When we choose which services to add as OAuth providers, we will
assess in what ways the candidate services also provide these protections if we want to continue to
have this benefit.

Once we have the code to check accounts with GitHub's API to see if they've been renamed or
deleted, we could proactively periodically run that code on accounts that haven't been used
recently to keep crates.io more accurate regardless of when people log in.

We could limit how often a user may change their username, to potentially limit cases where someone
is trying to impersonate someone or confuse others.

We could track history of username changes (starting from whenever the feature is implemented, we
don't have historical data) and display on a user's page their historical usernames and dates when
they were changed. This would be good for transparency but problematic in cases such as someone
transitioning and wanting to remove all association with their deadname (if their name is part of
their GitHub account). Perhaps this information could be retained and viewable by admins only.

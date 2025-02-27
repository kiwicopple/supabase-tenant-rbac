# Supabase Multi-Tenant Role-based Access Control

This is a template (set of db migrations) which attempts to provide a group and role system for supabase projects. It is based off the supabase community [custom claims work done here](https://github.com/supabase-community/supabase-custom-claims).

## Disclaimer

I built this for my own personal use. It has not been audited by an independent security agency nor has it been thouroughly tested beyond my own project. It has been slightly modified for public release so there may be bugs and other pitfalls. I highly encourage users to perform their own inspection of the security of this template. Please submit any issues you find and I will do my best to promptly resolve them. Use at your own risk

## Features

- Create groups of users
- Assign a user to a group
- Assign a role to the user within a group
- Users can have multiple roles in a single group
- Manage group and role assignments through a table
- Group and role memberships are stored in the group_user table, auth.users table, and JWT
- Create RLS policies based on the user's assigned group or role
- RLS policies can reference either the JWT (for performance) or the table (for security)
- Includes a user_role view which allows easy reference of which users have which roles
- user_role view can be used to delete roles via trigger
- (Optionally) Users can create their own groups
- Group owners can add other users to their groups and assign roles

## Drawbacks

As with the original [custom-claims repository](https://github.com/supabase-community/supabase-custom-claims), this also suffers from the [JWT token refresh issue](https://github.com/supabase-community/supabase-custom-claims#what-are-the-drawbacks-to-using-custom-claims). Put simply, when a user's group or role is changed, there is (currently) no performant way to force an active session token to refresh. This means that the user will continue to have their old group/role assignments until the session expires.

This particular project can be made to work around this a couple of ways:

- Use the non-jwt functions which access the roles table directly (this is less performant because it adds an extra table query to every RBAC-enabled RLS query)
- Allow users to subscribe to the roles table via realtime and have the client refresh the token when a change is detected

## Compared to [custom-claims](https://github.com/supabase-community/supabase-custom-claims)

The custom-claims project provides built-in functions for adding and removing generic claims to/from the `raw_app_meta_data` field of the auth.users records. This is highly flexible but is not opinionated about the way groups and roles should be handled.

Additionally, the custom-claims project requires the use of functions for all changes and does not provide the table-based interface that this project employs to maintain claims via trigger.

Overall, these projects are very similar and this one is built on top of the work that was done on the custom-claims project. This is merely an attempt to provide a more opinionated and streamlined solution that can be used by people who are not as familiar with RBAC or SQL functions.

## Under the hood

All group and role administration happens in the `groups` and `group_users` tables. The only exception is user role deletion, which can be performed through the `user_roles` view. When a record is inserted or changed in the `group_users` table, a function is triggered which detects the change and updates the corresponding `auth.users` record. Specifically, the `raw_app_meta_data` field is updated with data such as:

```json
{
  "groups": {
    "c2aa61f5-d86b-45e8-9e6d-a5bae98cd530": ["admin"]
  },
  "provider": "email",
  "providers": ["email"]
}
```

Notice that the `groups` key has been added by the trigger function and has been populated with a new subkey matching the uuid of the group that the user was added to. Finally, the value is an array of roles as strings. So the structure is `raw_app_meta_data -> groups -> group_id -> roles[]`

In this example, the `provider` and `providers` keys are a normal part of the supabase backend and are irrelevant to this project.

As a security note, `raw_app_meta_data` is stored within the JWTs when a session is created. This field is not editable by the user under normal circumstances.

## Installation

#### Pre-check

- Requires PostgreSQL 15.x (due to use of "security_invoker" on the user_role view)
- This creates the following tables / views. Make sure they do not collide with existing tables:
  - groups
  - group_users
  - user_roles (view)
- This creates the following functions. Please check for collisions:
  - has_group_role
  - is_group_member
  - jwt_has_group_role
  - jwt_is_group_member
  - jwt_is_expired
  - update_user_roles
  - delete_group_users
  - set_group_owner
  - add_group_user_by_email

#### Installation via the SQL console

1. Copy the contents of one or more of the [migration files](supabase/migrations/) in this repository
1. Paste the contents into the SQL console on your supabase dashboard and run it
1. Optionally, run a diff from your supabase cli to create a migration file capturing these changes

#### Installation via local migration file

If you use the supabase cli and have a local dev environment, you can copy the migration files from this repo into your `supabase/migrations/*` folder and rename them to reflect a more recent timestamp. Note that in order for supabase to apply the migrations, they must conform to the `<timestamp>_name.sql` format.

### Security / RLS

Out of the box, the tables created by this project have RLS enabled and set to reject all operations. This means that group membership and roles can only be assigned and administered by the database administrator/superuser. You may want to modify the RLS policies on the `groups` and `group_users` tables to enable users (such as users with an "admin" role) to modify certain group and role memberships based on their own membership.

One idea is to make use of [a trigger to allow any user who creates a group to automatically become the owner/admin of that group](supabase/migrations/20230914220613_auto_set_group_owner_on_creation.sql). Then, via their owner role, they can [add additional users/roles](supabase/migrations/20230914231642_allow_owners_to_add_users_to_groups.sql). Note that in order for these to work, you will need to create an RLS policy that allows users to create groups (insert rows into the `groups` table). Here is an example of one such policy which allows any authenticated user to create a new group:

```sql
create policy "Allow authenticated users to insert"
on "public"."groups"
as permissive
for insert
to authenticated
with check (true);
```

I've also recently created an invite system built on this and supabase edge functions which allows group owners to generate a token which other users can use to join their group with a specific pre-selected role. I have not yet extensively tested it but if you are interested in this code as well, please open an issue ticket and I'll clean it up and get it added.

## How to use

1. Create a record in the "groups" table
2. Create a record in the "group_users" table which links to the group and the user via the foreign key columns ("group_id" and "user_id", respectively)
3. Observe that the respective user record in auth.users has an updated `raw_app_meta_data` field which contains group and role information
4. (Optional) Use the built-in role checking functions for RLS
   - is_group_member
   - has_group_role
   - jwt_is_group_member
   - jwt_has_group_role
5. (Optional) Check the JWT signature and contents to determine roles on the client-side

### Built-in functions

#### is_group_member(group_id uuid)

Required inputs: `group_id` as a uuid
Returns: boolean

This will return true if the `group_id` exists within the `raw_app_meta_data->groups` field of the user's record in the auth.users table. Executing this function will perform a query against the auth.users table each time it is run.

#### has_group_role(group_id uuid, group_role text)

Required inputs: `group_id` as a uuid, `group_role` as text
Returns: boolean

This will return true if the `group_role` exists within the `raw_app_meta_data->groups->group_id` array of the user's record in the auth.users table. Executing this function will perform a query against the auth.users table each time it is run.

#### jwt_is_group_member(group_id uuid)

Required inputs: `group_id` as a uuid
Returns: boolean

This will return true if the `group_id` exists within the `app_meta_data->groups` field of the JWT token used to perform the request. Executing this function does not perform any database queries as it relies only on the JWT.

#### jwt_has_group_role(group_id uuid, group_role text)

Required inputs: `group_id` as a uuid, `group_role` as text
Returns: boolean

This will return true if the `group_role` exists within the `app_meta_data->groups->group_id` field of the JWT token used to perform the request. Executing this function does not perform any database queries as it relies only on the JWT.

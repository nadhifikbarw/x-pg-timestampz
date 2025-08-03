# "For the sake of timestamp"

## TLDR Recommendation
> Use `TIMESTAMPTZ` instead of `TIMESTAMP` as encoding time zone data is valuable. But always ensure session time zone is using UTC either via config or calling `SET TIME ZONE 'UTC';`
upon establishing DB connection. This allows users to worry less about losing time zone information if you have implementations that pass tz offset or don't expect timestamp to contain time zone offset.

## Background

Most common use case of using `timestamp` type for application development is to track `created_at`, `updated_at`, and `deleted_at` on each tables.

In case you've never heard, [postgresql discourages the use of `timestamp` type](https://wiki.postgresql.org/wiki/Don%27t_Do_This#:~:text=might%20be%20suitable.-,Don%27t%20use%20timestamp%20(without%20time%20zone)%20to%20store%20UTC%20times,-Storing%20UTC%20values) to store timestamps, and recommends `timestamp with time zone` a.k.a `timestamptz`.

Both `timestamp` and `timestamptz` are stored using 64-bit integer representing microsecond offset from 2000-01-01 for PostgreSQL.

> Historically, The SQL standard requires that writing just `timestamp` be equivalent to `timestamp without time zone`, and PostgreSQL honors that behavior. [...]

## `timestamptz` resolves time zone offset before storing

Confusingly, `timestamptz` doesn't actually store any timezone information whatsoever. Instead `timestamptz` instructs the database to consider time zone offset from input before storing information as UTC. So the typical "store time value in UTC" recommendation is still relevant here.

### Input with time zone offset

Both `timestamp` and `timestamptz` still stores value in UTC. But `timestamptz` will treat value such as `2004-10-19 10:23:54+02` and convert it as `2004-10-19 08:23:54Z` before storing such value. Once converted value is stored, `+02` information is no omitted and no longer relevant.

If you provide `2004-10-19 10:23:54+02` into `timestamp` column it will ignore `+02` part and store `2004-10-19 10:23:54`, leading to incorrect timestamp getting stored.

### Input without time zone offset

When dealing with input with no timezone offset `timestamp` will treat input such as  `2004-10-19 10:23:54` as is.

For `timestamptz`, if input doesn't contain any time zone offset information. It will treat input such as `2004-10-19 10:23:54` as timestamp from local time zone defined by `timezone` setting in `postgresql.conf`. Thankfully, [`postgres` docker image uses `UTC` timezone setting](#checking-postgres-image-default-timezone) by default.

For example `2004-10-19 10:23:54` will be treated as `2004-10-19 10:23:54+08` if my session uses `Asia/Singapore` either from `SET TIME ZONE` or `postgresql.conf`

Thus if you want to specify timestamp with zero offset, you can use `z` (`zulu`) as time zone offset. e.g. `2004-10-19 10:23:54zulu` or `2004-10-19 10:23:54z`

> When you open a connection to PostgreSQL, you will have a session between you (the client) and the database (the server). The session will contain a time zone, which you can set with the SET TIME ZONE command for using PostgreSQL.

## `timestamptz` displays timestamp in local time zone

Additional behavior of `timestamptz` is automatic conversion of timestamp value before displaying or returning query results based on SESSION TIME ZONE. Thus using the same example above `2004-10-19 10:23:54+02` it will be persisted as  `2004-10-19 08:23:54` physically, but returned as `2004-10-19 08:23:54Z` (notice the Z) assuming default session time zone are set to UTC. If session is set to use `Asia/Singapore` time zone. It will returns `2004-10-19 16:23:54+08` instead.

> Remember this behavior doesn't mean `timestamptz` stores any tz information, it just converts the stored UTC value using active session time zone context.

## Usage

Considering time zone offset handling of `timestamptz`, using this type encoding can be valuable. It provides assurance that any timestamps in queries containing TZ offset be handled appropriately. Let's say you're writing db logic manually without ORM or simply forgot to perform UTC conversion before sending insert query to database. If it's `timestamp` type, Postgresql will simply ignore time zone offset information and proceed with storing inaccurate information.

## Caveats & Middle Ground

Using `timestamptz` however comes with some minor caveats but can easily resolved:

* Set UTC as default session timezone for all connections, this to prevent unexpected conversion when sending timestamp input without time zone offset, [thankfully most default already use this.](#checking-postgres-image-default-timezone)
* Always specify time zone offset when inserting into `timestamptz`, for timestamp that has been converted to UTC, simply append "Z" to final value, making the value explicit and unaffected by session time zone setting.
* While most language nowadays already have well-equipped mechanisms to handle and manipulate timestamp containing time zone offset when scanning query results from database. If you want extra safety/unsure with you database defaults. You can execute `SET timezone = 'UTC'` once you established new session.

## Checking `postgres` image default timezone

Without supplying custom configuration via `postgres.conf`, check interactively, execute commands in order:

```sh
docker network create pg-check-tz-net
docker run --rm --name pg-check-tz -e POSTGRES_PASSWORD=password -d --network pg-check-tz-net postgres
docker run -it --rm --network pg-check-tz-net postgres psql postgresql://postgres:password@pg-check-tz
show timezone;
\q
# Cleanup
docker container stop pg-check-tz
docker network rm pg-check-tz-net
```

Should show

```
 TimeZone
----------
 Etc/UTC
(1 row)
```






# Notes on `mysql`

* MySQL doesn't have `timestamptz` only `timestamp`, value is stored as UTC
* MySQL `timestamp` doesn't support time zone offset. Input will be treated as value of current session time zone, converted into UTC before storing.
* MySQL `timestamp` data will be converted into current session time zone for retrieval
* Due to two points above, ensuring session time zone is set into 'UTC' almost feels like necessity for sane `timestamp` handling. And application must always convert all timestamp as UTC before inserting.

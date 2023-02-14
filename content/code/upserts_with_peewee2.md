---
title: "Upserts With PeeWee2"
date: 2023-02-06T22:09:57+02:00
tags: ['backend', 'database', 'ORM', 'legacy']
---
# AKA what do you do when you are missing a critical function?

So, I tried to find a solution of using an external database, which cannot receive queries by any API, but rather just allows users to download itself as an excel file.

In this case it was a DB containing a mapping between US ZipCodes* and US states.

This mapping changes every once in a while, and I wanted to allow users to grab a new .xls file and upload it to our systems.

An elegant solution to this problem was to avoid deleting the entire zipcode table, but rather upserting it, specifically aiming at replacing values upon conflict.

(I probably could've just replaced the entire table but where's the fun in that)

## HOWEVER, My PeeWee** version was PeeWee2

I could use `insert` with an `on_conflict` argument like the following example:

```python
 # e.g. new_zipcode = 12345; new_state = 'MA', new_central_address = 'Some St. 3'
def upsert_zipcode(new_zipcode, new_state, new_central_address):
    ZipCode.insert(zipcode=new_zipcode, state=new_state, central_address=new_central_address)
            .on_conflict(
                # Keep the existing, matching zipcode, but replace the state and central address
                preserve=[ZipCode.zipcode], 
                update={ZipCode.state: new_state, ZipCode.central_address : new_central_address})
            .execute()
```

But then I'd need to iterate on all the new zipcodes I want to add, which might take a while...

Another approach is to use `insert_many`, if I'm working with dictionaries or lists or dataframe, but alas; *PeeWee2* has no `on_conflict` argument!

PeeWee3 supports this but upgrading the version wasn't viable at that time.

## So what did I do?

I replaced the entire table with `insert_many`.

Sometimes, a fresh start to data, especially if it is read-only, is better than preserving existing data.

Yep, there's no happy end to this story.

---

\* I bet you didn't know, but ZipCode™ is uppercased like that since it is trademarked by the United States Postal Service, and ZIP stands for "Zip Improvement Plan".
Now you know.

** In case if you never heard of it, a [pretty neat ORM](https://docs.peewee-orm.com/)
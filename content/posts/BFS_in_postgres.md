+++
title = "BFS in Postgres for cycle detection"
date = "2022-06-05T22:39:47+02:00"
author = "Mifour"
authorTwitter = "" #do not include @
cover = ""
tags = ["SQL", "Postggres"]
keywords = ["SQL", "Postgres"]
description = "Implementing Breath First Search algorithm in Postgres for fast cycle detection"
showFullContent = false
readingTime = true
hideComments = false
+++

# BFS in Postgres for cycle detection
*Implementing Breath First Search algorithm in Postgres for fast cycle detection*  
  
*TLDR; The complete script is available on my [github]()*  
  
*Originnally, I used Django but, I try to keep this article framework agnostic because the problem and the solution can concern any framework.*  
  
## Why would you do that?
I agree, it sounds dumb at first.  
Why would anyone make their Pg run such an algorithm, that is rarely used in prod if one doesn't work with graph?  
  
At this time, I was doing classic backend web development with a Django web server, a Postgres database and interacting with frontend through an API. 
  
I had a table User like this:
```SQL
CREATE TABLE Users (
    id   UUID PRIMARY KEY,
    name varchar(40) NOT NULL
);

```
I wanted to allowed users to reference other users and be referenced by others for some kind of parenting system. 
An user could reference many other users and could be referenced by many other users. Users could add or remove references at anytime.  
But most importantly, it did not make sense for a user to reference another user that referenced (even indirectly) him/her at first. In other terms, there should be no reference cycles.  
  
The Reference Table is a many to many table which looks like this:
```SQL
CREATE TABLE Reference (
	id INTEGER PRIMARY KEY,
   parent_id UUID NOT NULL,
   child_id UUID NOT NULL,
   PRIMARY KEY(parent_id, child_id),
   CONSTRAINT fk_child
      FOREIGN KEY(child_id) 
	  REFERENCES users(id),
   CONSTRAINT fk_parent
      FOREIGN KEY(parent_id) 
	  REFERENCES users(id)
);
```
The equivalent django model is:
```python
# myapp.models.py
from uuid import uuid4

from django.db import models

class Users(models.Model):
	id = models.UUIDField(primary_key=True, default=uuid4)
	name = models.CharField(max_length=40)


class Reference(models.Model):
	parent = models.ForeignKey("Users", on_delete=models.CASCADE, related_name="parents")
	child = models.ForeignKey("Users", on_delete=models.CASCADE, related_name="children")

	class Meta:
		unique_together = [["parent_id", "child_id"]]

```
  
This is how graph theory enters the stage. This is basically a graph use case with our users being the nodes and the references being the vertices.  
*Technically speaking, this is a tree*  
  
Let's fill up the tables:  
```SQL
insert into users(id, name) values
	(gen_random_uuid(), 'Alice'),
	(gen_random_uuid(), 'Bob'),
	(gen_random_uuid(), 'Charlie'),
	(gen_random_uuid(), 'Dany')
RETURNING *;
```
| id | name |
| ----------- | ----------- |
| 0ae8c267-d306-4397-9f26-285f81fd5257 | Alice |
| 76d855df-48f0-41dd-b4d9-b6ebfea6e2cc | Bob |
| d39f4647-e333-4404-8cc9-078ae4e6d88e | Charlie |
| 59b80ba1-966d-4f54-aa1e-9d8397cf8678 | Dany |
(4 rows)  
  
```SQL
insert into reference(parent_id, child_id) values
	()
;
```
So in this context, I had a bunch of algorithms that required the assumption there would have been no graph cycles, but I had to ensure this was true.  
If there were a loop, the production server could get stuck in an infinite loop, timeout requests or even crash due the Out of Memory errors.    

## Cycle Detection
So, I had a cycle detection problem.  
One way to solve it is to start from somewhere on the graph, keep visiting unvisited neighbour nodes and check if you visited a node twice. If that true, there is a cycle. If no, the visited graph was cycle free.  
  
The algorithm I just described is pretty much the [Breath First Search algorithm](https://en.wikipedia.org/wiki/Breadth-first_search), or BFS for short. It is known since 1945 the and used in graph theory. Today, almost every computer science 101 books would have a chapter on it.  
  
## First idea
When the api server will receive a request to add a reference, it would run a bfs script like:
```python
from uuid import UUID

def cycle_detected(user, ref_to_other):
	visited = {UUID(user.id, version=4)}
	to_visit = [ref_to_other]
	while neighbours := set(
		Users.objects.filter(
			id__in=Reference.objects.filter(parent_id__in=to_visit).values_list('child_id', flat=True)
		).values_list('id', flat=True)
	):
		if visited & neighbours:
			return True  # detected a cycle
		visited &= neighbours
		to_visit = list(neighbours)
	return False  # no cycle was detected

```
before adding the many to many relation to the Reference table.  
  
That would work but, I don't like to call the db many times (and potentially a lot of times) only to know if I can then insert a row in db. Data sent through expensive connection and literally forgot a few milliseconds after. A pair of servers talking a little chunk of data and waiting for the other to respond, to deserialize the data, to serialize it again and again until the algorithm is over. I'm not even talking of concurrency issues if those requests are not isolated.    
That's just wasteful.  
  
The cause of this iteration of db calls is due to the fact the server doesn't have all the relations between users in memory (and we don't want to load all relations in memory). This combines poorly with the BFS algorithm that discover iteratively the neighbours. Putting a db call in each iteration of the algorithm is a terribly slow implementation at scale.  

## Better implementation with Postgres
If all we want is to return a boolean, then we can just move the function from the API server to DB server.  
Fortunately, Postgres let us declare functions we can call in later statements.  
This will be a lot faster because there will be no network calls interrupting the algorithm that can just run over the data that just sit on the disk right next to it.  
  
```SQL
--bfs.sql
create or replace function cycle_detected(visited UUID[], to_visit UUID[])
   returns boolean 
   language plpgsql
  as
$$
declare 
	children UUID[];
begin
	select array_agg(child_id) from reference where parent_id::UUID = ANY(to_visit) into children;
	return case when children && visited
		then true
		else case when array_length(children, 1) > 0
			then cycle_detected(visited || children, children)
			else false
		end
	end as cycle_detected;
end;
$$;
```
or
```SQL
create or replace function cycle_detected_loop(visited UUID[], to_visit UUID[])
	returns boolean
	language plpgsql
	as
$$
declare
	detected boolean;
	children UUID[];
begin
	select false into detected;
	select array_agg(child_id) from reference where parent_id::UUID = ANY(to_visit) into children;
	while !detected || array_length(children, 1) > 0 loop
		select children && visited into detected;
		select visited || children into visited;
		select children into to_visit;
		select array_agg(child_id) from reference where parent_id::UUID = ANY(to_visit) into children;
	end loop;
	return detected as cycle_detected;
end;
$$;
```
https://stackoverflow.com/questions/16992339/why-is-postgresql-array-access-so-much-faster-in-c-than-in-pl-pgsql


I did changed the algorithm from an iterative implementation into a recursive implementation but that does not changed the instructions too much, and in sql the recursive version has cleaner code (according to me).  

Let's check quickly it does work as intended:
```SQL
insert into users(id, name) values 
        ('0575d9c7-965b-4794-9391-aee2645fbbb1', 'James'),('1c504d03-c053-4850-8d52-2b1c0472bfbe', 'Mary'), ('5e174966-4a7a-412d-a8db-20da7842f62d', 'Robert');
insert into reference values ('0575d9c7-965b-4794-9391-aee2645fbbb1', '5e174966-4a7a-412d-a8db-20da7842f62d'), ('0575d9c7-965b-4794-9391-aee2645fbbb1', '1c504d03-c053-4850-8d52-2b1c0472bfbe');

select cycle_detected(array['5e174966-4a7a-412d-a8db-20da7842f62d'::uuid], array['0575d9c7-965b-4794-9391-aee2645fbbb1'::uuid]);
```
| cycle_detected |
| -------------- |
| t |
(1 row)  
  
```SQL
select cycle_detected(array['5e174966-4a7a-412d-a8db-20da7842f62d'::uuid], array['aaaaaaaa-aaaa-0000-0000-aaaaaaaaaaaa'::uuid]);
```
| cycle_detected |
| -------------- |
| f |
(1 row)  
  
All good! 
  

  
What is great also, is the fact I can call this function and use it in inside a procedure. That procedure can run at the start of the database server to ensure it is already in a valid state or, it run before every write operation on the References Table to ensure no one (even with different server than ours) did corrupt the table.  
```SQL
PROCEDURE;
```
  
In the API server, there will be some `try/except` block just in case someone did try to insert invalid data.  
```python
try:
	References.objects.create(parent=parent_user, child=child_user)
except SQLException as e:
	print(f"referencing from {parent_user} to {child_user}, would create a cycle\n{e}")
	return response(400, f"referencing from {parent_user} to {child_user} is invalid")
return response(201, "created")
```

## Declaring the Postgres function in migration
To ensure the function is declared in the database, the good practice is just to declare it through a migration:
```python
# migration.py
from django.db import migrations

with open("bsf.sql", "r") as file:
	sql_statement = file.read()

class Migration(migrations.Migration):

    dependencies = [
        ('yourappname', '0008_previous'),
    ]

    operations = [
    	migrations.RunSQL(
    		[
    			(
    				sql_statement,
    				None
  				)
  			]
    	)
    ]
```
## Benchmark 
In theory, it does save a lot of compute time.  
A single db call vs an undefined amount of shorts db calls over the network.  
  
Here's some benchmarks I measure to illustrate my point:  
This example checks for a cycle over a (hopefully) tree with a depth of x and each node having y branches (resulting of a tree with N^N-1 nodes approximately).  
All computations are done in local on my 2014 macbook air equipped with a duo core intel i5 and 4GB of ram. 

| program | depth x width | nb of users | time |
| ----------- | ----------- | ----------- | ----------- |
| server side computation | 2x2 | 3 | 5ms |
| db computation | 2x2 | 3 | 0.92 ms |
| ----------- | ----------- | ----------- | ----------- |
| server side computation | 5x5 | 781 | 45ms |
| db computation | 5x5 | 781 | 5.84 ms |
| ----------- | ----------- | ----------- | ----------- |
| server side computation | 10x3 | 30_000 | 875ms |
| db computation | 10x3 | 30_000 | 775.5 ms |
  
As you can see, the Postgres computation tends to be ... as fast?
Yes. This id mainly due to the Postgres algorimth uses an array not a set like the python version does.  
To improve yet again the Postgres version, we can use some pg tricks.  
  
## Pg extension intarray  
Up to this point I used the raw UUIDs from the user table but computing on this is a bit over-killed.
I can compute just as well by using unique integers that are the reference ids. This will let the possibility to leverage the built-in (Postgres extension IntArray)[https://www.postgresql.org/docs/current/intarray.html] which as a similar API and is supposed to perform much better. 
  
First, the computation will greatly improve using an index
```SQL
CREATE INDEX idx_reference_id ON reference(id);
```  
Installing the intarray extension can be done using
```SQL
CREATE EXTENSION intarray;
```
I did rewrite the `cycle_detected` function like this

I NEED USER IDS TO BE INT
```SQL
create or replace function cycle_detected_loop(visited integer[], to_visit integer[])
	returns boolean
	language plpgsql
	as
$$
declare
	detected boolean;
	children integer[];
begin
	select false into detected;
	select array_agg(child_id) from reference where parent_id::UUID = ANY(to_visit) into children;
	while !detected || array_length(children, 1) > 0 loop
		select children && visited into detected;
		select visited || children into visited;
		select children into to_visit;
		select array_agg(child_id) from reference where parent_id::UUID = ANY(to_visit) into children;
	end loop;
	return detected as cycle_detected;
end;
$$;
_____________
## Thanks

Thank you for reading this article. I hope you like it and get inspired to solve problems with Postgres.  
  
Have a nice day,  
Mifour :) 


time psql -d bfs -U postgres -c "select cycle_detected(array['84e75c6f-e86e-48f2-b488-efec646210ea'::uuid], array['5ce3a84e-e9eb-4e1d-aab7-f2f22a42f901'::uuid]);"
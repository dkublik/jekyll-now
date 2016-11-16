---
layout: post
title: Should I go Eager or Lazy with JPA mappings?
comments: true
---
The short answer is: go Lazy if you care about number of queries happening under the hood. _"But what if I always need a particular relation?"_ - still go lazy, here is why.


#### There is No Such Thing As a Free Lunch

I used to think: _"In many scenarios eager is for free. If in most queries you need a particular relation then it's better to fetch it eagerly in one join, and in those rare cases when don't need it - one additional join costs almost nothing"_.

An example would be a Worker, always going in pair with his Unit.


```java
@Entity
class Worker {
    @Id
    private String personalId;

    private String surname;

    @ManyToOne
    private Unit unit;
}

@Entity
class Unit {
    @Id
    private Long id;
}

interface WorkerRepository extends CrudRepository<Worker, String> {
} 
```  

By calling _workerRepository.findOne(personalId)_ I will produce


```sql 
select worker0_.personal_id as personal1_1_0_, worker0_.surname as surname2_1_0_, worker0_.unit_id as unit_id3_1_0_, unit1_.id as id1_0_1_ from worker worker0_ left outer join unit unit1_ on worker0_.unit_id=unit1_.id where worker0_.personal_id=?
```  


This is perfect. I got only one query (join on worker and unit tables) and I can access all _Unit_ information for free - without a need for additional query.
It's true - but only when querying by id.



#### The Problem with Eager Mapping

Imagine you need to query by surname - _findBySurname()_


```java
interface WorkerRepository extends CrudRepository<Worker, String> {
    Worker findBySurname(String surname);
}
```  

Now suddenly you we'got 2 queries:


```sql 
select worker0_.personal_id as personal1_1_0_, worker0_.surname as surname2_1_0_, worker0_.unit_id as unit_id3_1_0_ from worker worker0_ where worker0_.personal_id=?
select unit0_.id as id1_0_0_ from unit unit0_ where unit0_.id=?
```  


Two queries even if you'll never access _Worker.unit_ property. 


```java
def "should create two queries when calling by findBySurname"() {
	when:
		workerRepository.findBySurname('Eager')
	then:
		queryStatistics.nrOfQueries() == 2
}
```  
[source](https://github.com/dkublik/sd-fetching/blob/master/src/test/groovy/pl/dk/sdfetching/eager/EagerWorkerRepositorySpec.groovy)

It happens cause with the _findBySurname()_ you are asking only for the _Worker_ entity. Then - when the object is constructed - framework finds out that _Worker.unit_ property has an eager mapping - so need to be set, but since _Unit_ data is not present - one more query need to be performed.

The same will be true for JPQL:


```java
interface WorkerRepository extends CrudRepository<Worker, String> {
    @Query("select w from Worker w where w.surname = :surname")
    Worker findBySurnameJPQL(@Param("surname") String surname);
}
```  

&nbsp;
Since _Unit_ has eager mapping - to avoid additional select you will always need to fetch it manually.

+ By JPQL

```java
interface WorkerRepository extends CrudRepository<Worker, String> {
    @Query("select w from EagerWorker w join fetch w.unit u where w.surname = :surname")
    Worker findBySurnameJPQLFetchingUnit(@Param("surname") String surname);
}
```  

+ or by EntityGraph

```java
interface WorkerRepository extends CrudRepository<Worker, String> {
    @EntityGraph(attributePaths = "unit")
    Worker findBySurname(String surname);
}
```  

&nbsp;

#### Avoiding the Danger

So if you add an eager mapping - every time anyone adds new query method to your repository and forget to fetch all eager relations, additional query will be created for every eager mapping in your entity!

And from the other side:
For every query method in your repo - you will always need to manually fetch all eager relations - even when you don't need them.

Imagine having repo with 10 query methods - all fetching all eager relations. Now you need to modify your entity - if you add new eager relation,
you will need to add it to every existing query method to avoid additional join.
It doesn't end with one repository. Imagine _Unit_ having eager relation on it's own. Now you would need to add another fetch:

```java
interface EagerWorkerRepository extends CrudRepository<EagerWorker, String> {
    @Query("select w from Worker w join fetch w.unit u join fetch u.owner where w.surname = :surname")
    Worker findBySurname(@Param("surname") String surname);
}
```  

Somebody adding eager relation somewhere can cause perfomance issues in other repositories.


Of course you can use entity graphs. These can be reused - when being created in entity classes. But then you need to remember to add references over repository methods and they also polute entity classes. Moreover - they would need to repeat data in case when having eager relation in eager relation. (EntityGraph for _Unit_ need to have information to eagerly fetch _Owner_, and entity Graph for _Worker_ needs to have information to eagerly fetch _Unit_ and again _Owner_).

Better solution would be to set all your relations as LAZY.


#### Conclusion

Model is lazy, but you can still have eager queries - by JPQL. You still need to manually fetch your needed relations (not doing so will create as before additional query when in transaction or _LazyIntializationException_ when not).
But - when you don't need to access relation data - you are not forced to do fetching and no extra query is performed.
Moreover adding new Lazy relation shouldn't affect exisiting code.


To sum it up: I'm not saying that everything should be loaded lazy. The conclusion is: all entity mappings should be lazy and eager loading performed by explicit fetching.



Examples can be found: [here](https://github.com/dkublik/sd-fetching).

&nbsp;

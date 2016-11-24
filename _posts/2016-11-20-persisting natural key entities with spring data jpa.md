---
layout: post
title: Persisting Natural Key Entities with Spring Data JPA
comments: true
---
From the dawn of time there is an ongoing discussion about using [surrogate keys](https://en.wikipedia.org/wiki/Surrogate_key) vs [natural keys](https://en.wikipedia.org/wiki/Natural_key) in your database tables. 
I don't want to take any side here - just write about one consequence you'll face when persisting natural key entity with Spring Data repository.


From the perspective of this article the difference between a natural key and a surrogate key is that a natural key is assigned by an application (e.g. a personalId for a _Worker_, which can't be assigned by any generator strategy), whereas a surrogate key in many cases is assigned by a database (e.g. by a sequence or an identity table). Of course a surrogate key can be assigned by an application as well (like an application generated uuid) - these cases will also face problems that are mentioned below for natural key entities.

&nbsp;

#### The Difference in Number of Queries

If we create a _Worker_ with a surrogate key assigned by a database _(id)_


```java
class SurrogateKeyWorker {
    @Id
    @GeneratedValue(strategy = IDENTITY)
    private Long id;

    private String personalId;
}
```  

and try to persist it


```java
def "should save with one query for new object"() {
    given:
        SurrogateKeyWorker worker = new SurrogateKeyWorker(PERSONAL_ID);

    when:
        workerRepository.save(worker)

    then:
        queryStatistics.nrOfQueries() == 1
}
``` 

we'll end up with one generated query, which is exactly what was expected.

```sql
insert into surrogate_key_worker (id, personal_id) values (null, ?)
``` 


We might however decide that _Worker.id_ property is not needed and treat _personalId_ as an application assigned natural key.

```java
@Entity
class NaturalKeyWorker {
    @Id
    private String personalId;

    NaturalKeyWorker(String personalId) {
        this.personalId = personalId;
    }
}
``` 

Then - if we repeat our test - we'll observe that instead of one query we got two:

```sql
select naturalkey0_.personal_id as personal1_0_0_ from natural_key_worker naturalkey0_ where naturalkey0_.personal_id=?
insert into natural_key_worker (personal_id) values (?)
```


It's quate easy to find out that _org.springframework.data.jpa.repository.support.SimpleJpaRepository.save()_ method decides whether to persist or merge entity by checking if this entity is new or not.


```java
@Transactional
public <S extends T> S save(S entity) {
	if (entityInformation.isNew(entity)) {
		em.persist(entity);
		return entity;
	} else {
		return em.merge(entity);
	}
}
```

	
However _entityInformation.isNew(entity)_ is as simple as assuming that an entity is not new when it has not null id. This will be true for ids assigned by db but not for these assigned by an app. My _Worker_ got his id assigned on creation - long before he was persisted.

&nbsp;

#### Delivering isNew info

Of course with a framework like Spring there is a way to handle it. In [documentation](http://docs.spring.io/spring-data/jpa/docs/1.10.5.RELEASE/reference/html/#jpa.entity-persistence.saving-entites) we can read about options for detection whether an entity is new or not:


![isNew() detection]({{ site.baseurl }}/images/2016-11-20-natural-key-persist/isNewDetection.png "isNew() detection")


Since option 3 is discouraged (I don't think that natural key is that rare thing) let's try to go with 2nd.


We'll need to implement _Persistable_ interface and somehow decide if an entity is new or not. A very naive implementation might look like the one below - deciding by passing an additional argument. Notice that
when _Worker_ is obtained by repo, it is instiantiated by a default, no args constructor, so isNew = false;


```java
@Entity
class PersistableWorker implements Persistable {
    @Id
    private String personalId;

    @Setter
    private String surname;

    @Transient
    private boolean isNew = false;

    PersistableWorker(String personalId, boolean isNew) {
        this.personalId = personalId;
        this.isNew = isNew;
    }

    @Override
    public Serializable getId() {
        return personalId;
    }

    @Override
    public boolean isNew() {
        return isNew;
    }
}
```


If I rerun my test now I will got only one query persisting my object.

&nbsp;

#### Controlling the Flow

So why is _isNew()_ checked in _save()_ method in the first place? Well - if the object is not new then you don't want to insert but update it, so you need to retrieve it first like in the example below

```java
def "should save with two queries for existing object"() {
	given:
		PersistableWorker worker = new PersistableWorker(PERSONAL_ID, true)
		workerRepository.save(worker)
		queryStatistics.clear()
	when:
		worker = new PersistableWorker(PERSONAL_ID, false) // 1
		worker.surname = 'Smith' // 2
		workerRepository.save(worker) //3
	then:
		queryStatistics.nrOfQueries() == 2
}
```

```sql
select persistabl0_.personal_id as personal1_1_0_, persistabl0_.surname as surname2_1_0_ from persistable_worker persistabl0_ where persistabl0_.personal_id=?
update persistable_worker set surname=? where personal_id=?
```


The _When_ section shows scenario when

1. Existing worker is created (e.g. by data taken from obtained dto)
2. got his name changed
3. and merged to the database (select + insert)

It works fine, except this is a scenario I never face.

What I would do would be:


```java
def "should save with one query for existing object"() {
	given:
		PersistableWorker worker = new PersistableWorker(PERSONAL_ID, true);
		workerRepository.save(worker)
	when:
		doInTx {
			worker = workerRepository.findOne(PERSONAL_ID)
			queryStatistics.clear()
			worker.surname = 'Smith'
		}
	then:
		queryStatistics.nrOfQueries() == 1
		workerRepository.findOne(PERSONAL_ID).surname == 'Smith'
}
```


I don't need to call _save()_ if I retrieved my _Worker_ in the same transaction that I'm changing it, because _dirty checking_ mechanism will do that for me. So I never use _save()_ for an update operation - but only for an insert. If so - then instead of calling _save()_ and having the framework decide for me which operation to perform I would like be able call _persist()_ by myself.

There is no _persist()_ method in Spring Data _org.springframework.data.repository.CrudRepository_, but it's quite easy to add it.
What you need to do is instead of extending _CrudRepository_ with your repos, create your own base interface with a simple implementation.


```java
@NoRepositoryBean
public interface BaseRepository<T, ID extends Serializable> extends CrudRepository<T, ID> {
    <S extends T> S persist(S entity);
}

public class BaseRepositoryImpl<T, ID extends Serializable> extends SimpleJpaRepository<T, ID> implements BaseRepository<T, ID> {

    private final EntityManager entityManager;

    @SuppressWarnings("unchecked")
    public BaseRepositoryImpl(JpaMetamodelEntityInformation jpaMetamodelEntityInformation, Object o) {
        super(jpaMetamodelEntityInformation.getJavaType(), (EntityManager) o);
        this.entityManager = (EntityManager) o;
    }

    @Override
    @Transactional
    public <S extends T> S persist(S entity) {
        entityManager.persist(entity);
        return entity;
    }
}
```


and register it with _@EnableJpaRepositories(repositoryBaseClass = BaseRepositoryImpl.class)_


Now, extending _Persistable_ with _Worker_ is not needed, as _workerRepository.persist(worker)_ operation results only in one insert. All of that just makes me wonder - Why Spring Data JPA decided to hide _entityManager.persist()_ method in the first place?


Full Examples can be found: [here](https://github.com/dkublik/sd-natural-keys).

&nbsp;

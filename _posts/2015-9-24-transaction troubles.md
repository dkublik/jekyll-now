---
layout: post
title: Transaction Troubles
comments: true
---

Recently I've faced an interesting issue. Got transactional method saving entity to the database - method is called, no exception is thrown - but no data is stored into the db.

&nbsp;

```java
/* SummaryMaker */

    @EventListener
    @Transactional
    public void createSummary(FileProcessedEvent event) {
        summaryRepository.save(
                new Summary(event.getFileName(), event.getLinesProcessed())
        );
    }
```  

_@Transactional_ is from the Spring framework and repo is _spring-data-jpa_ - so even if there was not active transaction then _SimpleJpaRepository_ (_JpaRepository_ implementation) would create one.  
Moreover, I know everything is configured correctly since all the other transactional methods in the project work well.  
Is this transaction by any chance read-only? I check that quickly - and it appears it isn't.  
But debugger shows that my transaction is not new (there is an external transaction around it) - and this is my lead.

&nbsp;

So the expected flow in this case goes as follows:

1. _createSummary()_ method is called
2. method needs transaction (_@Transactional_), so one is created 
3. external transaction is present, so createSummary transaction (let's call it internal transaction) joins it, as propagation was not specified and default one is _Propagation.REQUIRED_
4. Only one commit is expected - the one from the external transaction.

![expected transaction flow]({{ site.baseurl }}/images//2015-09-2-transactions-trouble/trans1.png "expected transaction flow")

&nbsp;

This is clearly not what is happening in our case, but to understand it we need to find out how and when _createSummary()_ method is called.

&nbsp;

#### Context

1) The whole case is about some file processing. There is a _FileProcessor_ with _@Transactional_ _processFile()_ method, which publishes application event when processing is done.
 
```java
/* FileProcessor */

    @Transactional
    public void processFile(String fileName) {
        // do some processing
        eventPublisher.publish(new FileProcessedEvent(this, fileName, linesProcessed));
    }
```  

&nbsp;

2) _EventPublisher_ uses _spring_ _ApplicationEventPublisher_ but does a little more. Someone figured out that it will check if a transaction is present and

+ if it is - it will publish after commit
+ if not - it will publish immediately  

I can think about three reasons for such a solution:

+ to make the transaction as quick as possible
+ to make sure all the data are already in db, since event may be handled outside the scope of current transaction
+ to make sure that we trigger event only when everything went ok (data is in db, no rollback occurred)
		
this is how it looks in the code:
	
```java
/* EventPublisher */

	public void publish(ApplicationEvent event) {
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            publishAfterCommit(event);
        } else {
            publishNonTransactional(event);
        }
    }

    private void publishAfterCommit(final ApplicationEvent event) {
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
                publishNonTransactional(event);
            }
        });
    }

    private void publishNonTransactional(ApplicationEvent e) {
        applicationEventPublisher.publishEvent(e);
    }
```  

&nbsp;
	
3) _FileProcessedEvent_ is handled by _SummaryMaker.createSummary(FileProcessedEvent event)_ - the method we've already seen.

The _createSummary()_ method needs a transaction as well, so here is what is expected:

+ _processFile()_ publishes an event
+ _EventPublisher_ sees that there is an active transaction (_createSummary()_ is not finished yet) so it waits for the commit.
+ after the commit, when transaction is finished, event is published
+ event is handled by _SummaryMaker.createSummary()_ in the scope of a new transaction.

And of course again - this is not what is happening.

&nbsp;

#### What is happening

The error assumption is that transaction ends immediately after a commit, when in fact, it isn't.

To be notified about commit we registered _TransactionSynchronization_.
If we examine _org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(DefaultTransactionStatus status)_
we will see the following steps:

+ before commit actions
+ actual commit
+ after commit actions - where our registered synchronization is called and event is fired.
+ cleanup - where transaction is finished (removed from thread local and stops being active)

so what we achieved with our code was:

![actual transaction flow]({{ site.baseurl }}/images//2015-09-2-transactions-trouble/trans2.png "actual transaction flow")

&nbsp;

1. external transaction commits
2. internal transaction joins the external one
3. internal transaction does not commit cause the external transaction is active
4. external transaction is finished

&nbsp;

#### Solution

Understanding what happened, we now can see that the solution is simple - we shouldn't join the existing transaction,
and to achieve it we only need to change propagation to *REQUIRES_NEW* in _SummaryMaker_

```java
/* SummaryMaker */

	@EventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void createSummary(FileProcessedEvent event) {
        summaryRepository.save(
                new Summary(event.getFileName(), event.getLinesProcessed())
        );
    }
```  

![corrected transaction flow]({{ site.baseurl }}/images//2015-09-2-transactions-trouble/trans3.png "corrected transaction flow")

&nbsp;

You can check the demo code ["here"](https://github.com/dkublik/transaction-troubles)

&nbsp;
****




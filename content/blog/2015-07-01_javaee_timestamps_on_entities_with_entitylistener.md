---
author: "Jørgen Ringen"
title: "JavaEE: Timestamps on entities with entitylistener"
date: 2015-07-01
slug: javaee_timestamps_entities_entitylistener
---

This post shows how you can add timestamps to your entities in a non-intrusive unified way with plain JPA.

---

We start off with an embeddable JPA entity, TimeStamp:

```java
import javax.persistence.Column;
import javax.persistence.Embeddable;
import java.util.Date;

@Embeddable
public class TimeStamp {

    @Column(name = "date_created")
    @Temporal(TemporalType.TIMESTAMP)
    private Date created;

    @Column(name = "date_updated")
    @Temporal(TemporalType.TIMESTAMP)
    private Date updated;

    // getters, setters, etc
}
```

We also need a simple interface that our entities should implement:

```java
public interface EntityWithTimeStamp {

    void setTimeStamp(TimeStamp timeStamp);
    TimeStamp getTimeStamp();
}
```

Lets add an entity which implements our interface:

```java
import javax.persistence.Column;
import javax.persistence.Embedded;
import javax.persistence.Entity;
import javax.persistence.EntityListeners;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

@Entity
@EntityListeners(TimeStampListener.class)
public class Person implements EntityWithTimeStamp {

    @Id
    @GeneratedValue
    private Long id;

    @Column
    private String name;

    @Embedded
    private TimeStamp timeStamp;

    // getters, setters, etc

    @Override
    public void setTimeStamp(TimeStamp timeStamp) {
        this.timeStamp = timeStamp;
    }

    @Override
    public TimeStamp getTimeStamp() {
        return this.timeStamp;
    }
}
```

As you can see, the entity needs three things to make it "timestampable":

* Implement our timestamp-interface
* Declare an embedded TimeStamp field
* Use the EntityListener annotation with a callback listener class

---

Our EntityListener class is just a simple java class with annotated methods declaring which events from the JPA runtime it should perform actions upon:

```
public class TimeStampListener {

    @PrePersist
    public void persist(Object entity) {
        if (entity instanceof EntityWithTimeStamp) {
            TimeStamp timeStamp = new TimeStamp();
            timeStamp.setCreated(new Date());
            ((EntityWithTimeStamp) entity).setTimeStamp(timeStamp);
        }
    }

    @PreUpdate
    public void updated(Object entity) {
        if (entity instanceof EntityWithTimeStamp) {
            TimeStamp existingTimeStamp = ((EntityWithTimeStamp) entity).getTimeStamp();
            existingTimeStamp.setUpdated(new Date());
        }
    }
}
```

The listener will be used by all our entities which should contain a TimeStamp field.

An EntityListener can easily be expanded with methods @PreRemove, @PostPersist, @PostUpdate, etc. The annotations can also be put directly on the entity which means that you don't need the EntityListener at all.

---

Simple Test:
```java
@Test
public void testTimeStamp() throws InterruptedException {
    // create
    tx.begin();
    Person person = new Person();
    person.setName("Name1");
    person = em.merge(person);
    tx.commit();

    assertThat(person.getTimeStamp().getCreated(), not(nullValue()));
    assertThat(person.getTimeStamp().getUpdated(), is(nullValue()));

    Thread.sleep(5);

    // update
    tx.begin();
    person = em.find(Person.class, person.getId());
    person.setName("Name2");
    tx.commit();

    // verify timestamp
    assertThat(person.getTimeStamp().getCreated(), not(nullValue()));
    assertThat(person.getTimeStamp().getUpdated(), not(nullValue()));
    assertThat(person.getTimeStamp().getUpdated(), after(person.getTimeStamp().getCreated()));
}
```

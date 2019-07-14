---
title: "Connecting the Dots, Data in Event Sourcing and Dimensional Modeling, How Close are They?"
header:
  teaser: /assets/images/20190714_events.png
  og_image: /assets/images/20190714_events.png
date: 2019-07-14
category: data-engineering
tags: [thought, event-sourcing, data-modeling, dimensonal-modeling, star-schema, architecture]
---

Working on both event sourcing and dimensional modeling, looking at the concept and understanding some use cases, I feel they are somehow similar. They both care about state changes, not the state itself. This makes me wonder, how close are they actually, can they work together in a, let say, hypothetical use case?

Let's start the thought process with a simple case: an online book store. The online book store allows you to order a book, proceed to checkout, fill payment details, and then the book will be delivered to you. To simplify the case, I won't use cart concept, only one book per order. Let say I have a service to handle the states of the book order, starting from ordered, paid, and delivered.

### Event Sourcing Design of the Online Book Store
It would be overkill to design event sourcing for this case of course, but let say we design the events, how the design will look like? Quoting Martin Fowler, Event Sourcing [1]:
> ensures that all changes to application state are stored as a sequence of events

Using event sourcing concept, instead of only storing the latest state of book order, I will store the state changes. This means, for the case above, I'll store several events: `Order Event`, `Pay Success Event`, and `Deliver Event`. Let's not go to infrastructure detail, let say I store it somewhere and apply state changes by subscribing to the events accordingly.

Here's the state I'll store for customer to see their current order status.
<figure class="third center">
  <img src="/assets/images/20190714_ordertable.png">
  <figcaption>Order Table</figcaption>
</figure>

And here's the events I will store in order to be able to replay the state, if I need to.
<figure class="half">
  <img src="/assets/images/20190714_events.png">
  <figcaption>Events</figcaption>
</figure>

### Dimensional Model Design of the Online Book Store
Now, what about dimensional model of the use case? Before we go into the design, here are dimensional modeling steps suggested by Kimball [2]:
1. Select business process
2. Decide granularity
3. Identify dimension
4. Identify facts and measures

To make things clearer to the use case, I'll pick one simple business question to test whether the dimensional model fit the purpose:
> I want to see daily count of books sold for the last 3 months grouped by book category

Let say, the definition of sales is when the payment is succeed. Then, the sales business process will use payment data, in combination with order data to get more context on what actually being paid.

1. Select business process: Sales
2. Decide granularity: Order Item because I need book category. However, as only one book allowed per order, order item grain = sales grain
3. Identify dimension: Book category, date
4. Identify facts and measures: Book quantity

See simplified dimensional design below (assume dim book is using SCD type 1, overwrite only latest state):
<figure class="half">
  <img src="/assets/images/20190714_factdim.png">
  <figcaption>Fact and Dimension</figcaption>
</figure>

The query to answer business question above will be:
```sql
SELECT
  sales_date,
  dim_book.category,
  SUM(quantity)
FROM
  fact_sales JOIN dim_book ON fact_sales.book_id = dim_book.book_id
WHERE
  sales_date > DATE_SUB(CURRENT_DATE(), INTERVAL 3 MONTH)
```

### So... How close?
Assuming I'm building a dimensional model data from event sourcing data, from the example above, there are several things that I can conclude:
* They both care about events, hence storing all events will make dimensional modeling a lot easier, as opposed to using change data capture or guessing state changes from the latest state in database.
* I might need to join data from several events in order to get dimensional model I want. In the example above, Pay Success Event is the definiton of sales, but it does not care about book detail. So I have to join `Pay Success Event` and `Order Event` in order to get Fact Sales table
* Not covered in the example above, but building a dimension table will need some events as well, such as `Add Book Event`.

Yet, this is hypothetical case only, the devils are in the details. Let me know if you see the conclusion otherwise, or you see my understanding of the concept is not accurate.

### References
* [1] https://martinfowler.com/eaaDev/EventSourcing.html
* [2] https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/four-4-step-design-process/
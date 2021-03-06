﻿
### Handling document relationships

One of the design principles that RavenDB adheres to is the idea that documents are independent, meaning that all data required to process a document is stored within the document itself. However, this does not mean that there should not be relations between objects.

There are valid scenarios where we need to define relationships between objects. By doing so, we expose ourselves to one major problem: whenever we load the containing entity, we are going to need to load data from the referenced entities too (unless we are not interested in them). While the alternative of storing the whole entity in every object graph it is referenced in seems cheaper at first, this proves to be quite costly in terms of database resources and network traffic.

RavenDB offers three elegant approaches to solve this problem. Each scenario will need to use one or more of them. When applied correctly, they can drastically improve performance, reduce network bandwidth and speedup development.

These are:

* Denormalization
* Includes
* Live Projections

The concepts behind this topic and other related subjects are discussed in length in the Theory section.

#### Denormalization

The easiest solution is to denormalize the data within the containing entity, forcing it to contain the actual value of the referenced entity in addition (or instead) of the foreign key.

Take this JSON document for example:

    { // Order document with id: orders/1234
        "Customer": {
            "Name": "Itamar",
            "Id": "customers/2345"
        },
        Items: [
            { 
                "Product": { 
                    "Id": "products/1234",
                    "Name": "Milk",
                    "Cost": 2.3
                },
                "Quantity": 3
            }
        ]
    }

As you can see, the `Order` document now contains denormalized data from both the `Customer` and the `Product` documents, which are saved elsewhere in full. Note we will not have copied all the customer properties into the order; instead we just clone the ones that we care about when displaying or processing an order. This approach is called *denormalized reference*.

The denormalization approach avoids many cross document lookups and results in only the necessary data being transmitted over the network, but it makes other scenarios more difficult. For example, consider the following entity structure as our start point:

    public class Order
    {
         public string CustomerId { get; set; }
         public string[] SupplierIds { get; set; }
         public Referral Refferal { get; set; }
         public LineItem[] LineItems { get; set; }
         public double TotalPrice { get; set; }
    }
   
    public class Customer
    {
        public string Name { get; set; }
        public string Address { get; set; }
        public short Age { get; set; }
        public string HashedPassword { get; set; }
    }

If we know that whenever we load an `Order` from the database we will need to know the customer's name and address, we could decide to create a denormalized `Order.Customer` field, and store those details in the directly in the `Order` object. Obviously, the password and other irrelevant details will not be denormalized:

    public class DenormalizedCustomer
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Address { get; set; }
    }

There will not be a direct reference between the `Order` and the `Customer`. Instead, `Order` holds a `DenormalizedCustomer`, which contains the interesting bits from `Customer` that we need whenever we process `Order` objects.

But, what happens when the user's address is changed? We will have to perform an aggregate operation to update all orders this customer has made. And what if the customer has a lot of orders or changes their address frequently? Keeping these details in sync could become very demanding on the server. What if another process that works with orders needs a different set of customer properties? The `DenormalizedCustomer` will need to be expanded, possibly to the point that the majority of the customer record is cloned.

Denormalization is a viable solution for rarely changing data, or for data that must remain the same despite the underlying referenced data changing over time.

#### Includes

The RavenDB "Includes" feature addresses the limitations of denormalization. Instead of one object containing copies of the properties from another object, it is only necessary to hold a reference to the second object. Then RavenDB can be instructed to pre-load the referenced document at the same time that the root object is retrieved. We can do this using:

    var order = session.Include<Order>(x => x.CustomerId)
        .Load("orders/1234");
   
    // this will not require querying the server!
    var cust = session.Load<Customer>(order.CustomerId);

Above we are asking RavenDB to retrieve the `Order` "orders/1234" and at the same time "include" the `Customer` referenced by the `Order.CustomerId` property. The second call to `Load()` is resolved completely client side (i.e. without a second request to the RavenDB server) because the relevant `Customer` object has already been retrieved (this is the full `Customer` object not a denormalized version). 

There is also a possibility to load multiple documents:

    var orders = session.Include<Order>(x => x.CustomerId)
        .Load("orders/1234", "orders/4321");
   
    foreach (var order in orders)
    {
        // this will not require querying the server!
        var cust = session.Load<Customer>(order.CustomerId);
    }

You can also use Includes with queries:

    var orders = session.Query<Order>()
        .Customize(x => x.Include<Order>(o => o.CustomerId))
        .Where(x => x.TotalPrice > 100)
        .ToList();
   
    foreach (var order in orders)
    {
        // this will not require querying the server!
        var cust = session.Load<Customer>(order.CustomerId);
    }

Under the hood, this works because RavenDB has two channels through which it can return information in response to a load request. The first is the Results channel, through which the root object retrieved by the `Load()` method call is returned. The second is the Includes channel, through which any included documents are sent back to the client. Client side, those included documents are not returned from the `Load()` method call, but they are added to the session unit of work, and subsequent requests to load them are served directly from the session cache, without requiring any additional queries to the server.

##### One to many includes

Include can be used with a many to one relationship. In the above classes, an `Order` has a property `SupplierIds` which contains an array of references to `Supplier` documents. The following code will cause the suppliers to be pre-loaded:

    var order = session.Include<Order>(x => x.SupplierIds)
        .Load("orders/1234");
   
    foreach (var supplierId in order.SupplierIds)
    {
        // this will not require querying the server!
        var supp = session.Load<Supplier>(supplierId);
    }

Again, the calls to `Load()` within the `foreach` loop will not require a call to the server as the `Supplier` objects will already be loaded into the session cache.

Multi-loads are also possible:

    var orders = session.Include<Order>(x => x.SupplierIds)
        .Load("orders/1234", "orders/4321");
   
    foreach (var order in orders)
    {
        foreach (var supplierId in order.SupplierIds)
        {
            // this will not require querying the server!
            var supp = session.Load<Supplier>(supplierId);
        }
    }

##### Secondary level includes

An Include does not need to work only on the value of a top level property within a document. It can be used to load a value from a secondary level. In the classes above, the `Order` contains a `Referral` property which is of the type:

    public class Referral
    {
        public string CustomerId { get; set; }
        public double CommissionPercentage { get; set; }
    }

This class contains an identifier for a `Customer`. The following code will include the document referenced by that secondary level identifier:

    var order = session.Include<Order>(x => x.Refferal.CustomerId)
        .Load("orders/1234");
   
    // this will not require querying the server!
    var referrer = session.Load<Customer>(order.Refferal.CustomerId);

Alternative way is to provide string based path:

    var order = session.Include("Refferal.CustomerId")
        .Load<Order>("orders/1234");
   
    // this will not require querying the server!
    var referrer = session.Load<Customer>(order.Refferal.CustomerId);

This secondary level include will also work with collections. The `Order.LineItems` property holds a collection of `LineItem` objects which each contain a reference to a `Product`:

    public class LineItem
    {
        public string ProductId { get; set; }
        public string Name { get; set; }
        public int Quantity { get; set; }
        public double Price { get; set; }
    }

The `Product` documents can be included using this syntax:

    var order = session.Include<Order>(x => x.LineItems.Select(li => li.ProductId))
        .Load("orders/1234");
   
    foreach (var lineItem in order.LineItems)
    {
        // this will not require querying the server!
        var product = session.Load<Product>(lineItem.ProductId);
    }

when you want to load multiple documents.

The `Select()` within the Include tells RavenDB which property of secondary level objects to use as a reference.

##### ValueType identifiers

The above `Include` samples assume that the Id property being used to resolve a reference is a string and it contains the full identifier for the referenced document (e.g. the `CustomerId` property will contain a value such as `"customers/5678"`). Include can also work with Value Type identifiers. Given these entities:

    public class Order2
    {
        public int Customer2Id { get; set; }
        public Guid[] Supplier2Ids { get; set; }
        public Referral2 Refferal2 { get; set; }
        public LineItem2[] LineItem2s { get; set; }
        public double TotalPrice { get; set; }
    }
   
    public class Customer2
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public string Address { get; set; }
        public short Age { get; set; }
        public string HashedPassword { get; set; }
    }
   
    public class Referral2
    {
        public int Customer2Id { get; set; }
        public double CommissionPercentage { get; set; }
    }
   
    public class LineItem2
    {
        public Guid Product2Id { get; set; }
        public string Name { get; set; }
        public int Quantity { get; set; }
        public double Price { get; set; }
    }

The samples above can be re-written as follows:

    var order = session.Include<Order2, Customer2>(x => x.Customer2Id)
        .Load("orders/1234");
     
    // this will not require querying the server!
    var cust2 = session.Load<Customer2>(order.Customer2Id);
    var orders = session.Query<Order2>()
        .Customize(x => x.Include<Order2, Customer2>(o => o.Customer2Id))
        .Where(x => x.TotalPrice > 100)
        .ToList();
     
    foreach (var order in orders)
    {
        // this will not require querying the server!
        var cust2 = session.Load<Customer2>(order.Customer2Id);
    }
    var order = session.Include<Order2, Supplier2>(x => x.Supplier2Ids)
        .Load("orders/1234");
     
    foreach (var supplier2Id in order.Supplier2Ids)
    {
        // this will not require querying the server!
        var supp2 = session.Load<Supplier2>(supplier2Id);
    }
    var order = session.Include<Order2, Customer2>(x => x.Refferal2.Customer2Id)
        .Load("orders/1234");
     
    // this will not require querying the server!
    var referrer2 = session.Load<Customer2>(order.Refferal2.Customer2Id);
    var order = session.Include<Order2, Product2>(x => x.LineItem2s.Select(li => li.Product2Id))
    .Load("orders/1234");
     
    foreach (var lineItem2 in order.LineItem2s)
    {
        // this will not require querying the server!
        var product2 = session.Load<Product2>(lineItem2.Product2Id);
    }

The second parameter to the generic `Include<T, TInclude>` specifies which document collection the reference is pointing to. RavenDB will combine the name of the collection with the value of the reference property to find the full identifier of the referenced document. For example, from the first example, if the value of the `Order.Customer2Id` property is the integer 56, RavenDB will include the document with an Id of "customer2s/56" from the database. The `Session.Load<Customer2>()` method will be passed the value 56 and will look for then load the document "customer2s/56" from the session cache.

##### Lucene Queries

Same query extensions have been applied to LuceneQuery, so to include `Customer` by `CustomerId` property can be achieved in both ways:

    var orders = session.Advanced.LuceneQuery<Order2>()
        .Include(x => x.Customer2Id)
        .WhereGreaterThan(x => x.TotalPrice, 100)
        .ToList();
   
    foreach (var order in orders)
    {
        // this will not require querying the server!
        var cust2 = session.Load<Customer2>(order.Customer2Id);
    }

or

    var orders = session.Advanced.LuceneQuery<Order2>()
        .Include("CustomerId")
        .WhereGreaterThan(x => x.TotalPrice, 100)
        .ToList();
     
    foreach (var order in orders)
    {
        // this will not require querying the server!
        var cust2 = session.Load<Customer2>(order.Customer2Id);
    }

##### Include rules

When using string-based includes like:

    var order = session.Include("Refferal.CustomerId")
        .Load<Order>("orders/1234");
     
    // this will not require querying the server!
    var referrer = session.Load<Customer>(order.Refferal.CustomerId);

you must remember to follow certain rules that must apply to the provided string path:

1.	**Dots** are used to separate properties e.g. `"Referral.CustomerId"` in example above means that our `Order` contains property `Referral` and that property contains another property called `CustomerId`.
2.	**Commas** are used to indicate that property is a collection type e.g. List. So if our `Order` will have a list of LineItems and each `LineItem` will contain `ProductId` property then we can create string path as follows: `"LineItems.,ProductId"`.
3.	**Prefixes** are used to indicate id prefix for **non-string** identifiers. e.g. if our `CustomerId` property in `"Referral.CustomerId"` path is an integer then we should add `customers/` prefix so the final path would look like `"Referral.CustomerId(customers/)"` and for our collection example it would be `"LineItems.,ProductId(products/)"` if the `ProductId` would be a non-string type.

{NOTE **Prefix** is not required for string identifiers, because they contain it by default. /}

Learning string path rules may be useful when you will want to query database using HTTP API.

   > curl -X GET "http://localhost:8080/queries/?include=LineItems.,ProductId(products/)&id=orders/1"

#### Live Projections

Using Includes is very powerful, but sometimes we want to do even more complex manipulation. The Live Projection feature is unique to RavenDB, and it can be thought of as the third step of the Map/Reduce operation: after having mapped all data, and it has been reduced (if the index is a Map/Reduce index), the RavenDB server can additionally transform the results into a completely different data structure and return that back instead of the original results.

Using the Live Projections feature, you have more control over what to load into the result entity, and since it returns a projection of the original entity, you also get the chance to filter out properties you do not need.

Let's look at an example to show how it can be used. Assuming we have many `User` entities and many of them are actually an alias for another user. If we wanted to display all users with their aliases using `Include()`, we would probably need to write something like this:

    // Storing a sample entity
    var entity = new User { Name = "Ayende" };
    session.Store(entity);
    session.Store(new User { Name = "Oren", AliasId = entity.Id });
    session.SaveChanges();
     
    // ...
    // ...
     
    // Get all users, mark AliasId as a field we want to use for Including
    var usersWithAliases = from user in session.Query<User>().Include(x => x.AliasId)
                           where user.AliasId != null
                           select user;
     
    var results = new List<UserAndAlias>(); // Prepare our results list
    foreach (var user in usersWithAliases)
    {
        // For each user, load its associated alias based on that user Id
        results.Add(new UserAndAlias
        {
            UserName = user.Name,
            Alias = session.Load<User>(user.AliasId).Name
        }
            );
    }

Since we use Includes, the server will only be accessed once - which is good, but the entire object graph for each referenced document (user entity for the alias) will be returned by the server... and it's an awful lot of code to write too!

Using Live Projections, we can get the same end result much more easily and with the transformation applied on the server side. This code defines an index which performs a Live Projection:

    public class Users_ByAlias : AbstractIndexCreationTask<User>
    {
        public Users_ByAlias()
        {
            Map = users => from user in users
                           select new { user.AliasId };
     
            TransformResults =
                (database, users) => from user in users
                                     let alias = database.Load<User>(user.AliasId)
                                     select new
                                                {
                                                    Name = user.Name,
                                                    Alias = alias.Name
                                                };
        }
    }

The function declared in `TransformResults` will be executed on the results of the query, which gives it the opportunity to modify, extend or filter those results. In this case, it lets us look at data from another document and use it to project a new return type.

A Live Projection will return a projection, on which you can use the `.As<>` clause to convert it back to a type known by your application:

    var usersWithAliases =
        (from user in session.Query<User, Users_ByAlias>()
         where user.AliasId != null
         select user).As<UserAndAlias>();

The main benefits of using Live Projections are; not having to write as much code, they run on the server and they reduce the network bandwidth by returning only the data we are interested in.

{NOTE An important difference to note is that while Includes are useful for both explicit loading by id and for querying, Live Projections can only be used for querying. /}

#### Combining Approaches

It is possible to combine the above techniques. Using the `DenormalizedCustomer` from above and creating an order that uses it:

    public class Order3
    {
        public DenormalizedCustomer Customer { get; set; }
        public string[] SupplierIds { get; set; }
        public Referral Refferal { get; set; }
        public LineItem[] LineItems { get; set; }
        public double TotalPrice { get; set; }
    }

We have the advantages of a denormalization, a quick and simple load of an `Order` and the fairly static `Customer` details that are required for most processing. But we also have the ability to easily and efficiently load the full `Customer` object when necessary using:

    var order = session.Include<Order3, Customer2>(x => x.Customer.Id)
        .Load("orders/1234");
     
    // this will not require querying the server!
    var fullCustomer = session.Load<Customer2>(order.Customer.Id);

This combining of denormalization and Includes could also be used with a list of denormalized objects.

It is possible to use Include on a query against a Live Projection. Includes are evaluated after the `TransformResults` has been evaluated. This opens up the possibility of implementing Tertiary Includes (i.e. retrieving documents that are referenced by documents that are referenced by the root document). 

Whilst RavenDB can support Tertiary Includes, before resorting to them you should re-evaluate your document model. Needing to use Tertiary Includes can be an indication that you are designing your documents along "Relational" lines.

#### Summary

There are no strict rules as to when to use which approach, but the general idea is to give it a lot of thought, and consider the implications each approach has.

As an example, in an e-commerce application it might be better to denormalize product names and prices into an order line object, since you want to make sure the customer sees the same price and product title in the order history. But the customer name and addresses should probably be references, rather than denormalized into the order entity.

For most cases where denormalization is not an option, Includes are probably the answer. Whenever serious processing is required after the Map/Reduce work is done, or when you need a different entity structure returned than those defined by your index - take a look at Live Projections.
﻿
### Using Linq to query RavenDB indexes

As we have already seen in the previous chapter, querying is made on a collection and using the `Session` object. Unless the user specifies what index to query explicitly, RavenDB will find an appropriate index to query, or create it on the fly if one does not already exist. We will see how to create named indexes, also called static, later in this chapter. Regardless of the index being actually queried, querying is usually done using Linq.

The built-in Linq provider implements the `IQueryable` interface in full. Its job is to automatically translates any Linq query into a Lucene query understandable by the RavenDB server, handling all the low-level details for you. Therefore, querying RavenDB is easy to anyone who has used Linq before; if you're not familiar with Linq, the following is a brief tutorial on how to use it to query RavenDB.

Assuming we have entities of the following types stored in our database:

	public class Employee
	{
	    public string Name { get; set; }
	    public string[] Specialties { get; set; }
	    public DateTime HiredAt { get; set; }
	    public double HourlyRate { get; set; }
	}
	 
	public class Company
	{
	    public string Id { get; set; }
	    public string Name { get; set; }
	    public List<Employee> Employees { get; set; }
	    public string Country { get; set; }
	    public int NumberOfHappyCustomers { get; set; }
	}

Let's see how we can use Linq to easily query for data in various scenarios. We will not cover all Linq's features, but will highlight several which are important to be familiar with.

#### Some basics

Say we wanted to get all entities of type `Company` into a List, how would we go about that? Ignoring for now the performance hit of loading a large bulk of data, we could do this pretty easily (note that this query is implicitly using Take(128), because it didn't specify a page size explicitly):

    var results =
    (
        from company in session.Query<Company>()
        select company
    ).ToArray();

Since the whole idea of using Linq here is to apply some filtering in an efficient manner, we can use Linq's `where` clause to do this for us:

	// Filtering by string comparison on a property
	var results = from company in session.Query<Company>()
	              where company.Name == "Hibernating Rhinos"
	              select company;
	 
	// Numeric property range
	results = from company in session.Query<Company>()
	          where company.NumberOfHappyCustomers > 100
	          select company;
	 
	// Filtering based on a nested (calculated) property
	results = from company in session.Query<Company>()
	          where company.Employees.Count > 10
	          select company;

Note, however, Linq is just a syntactic sugar. Under the hood, all queries are transformed into a sequence of method calls and lambda expressions. For example, the above code snippet is re-written by the compiler to look like the following before compiling:

	// Filtering by string comparison on a property
	var results = session.Query<Company>()
	    .Where(x => x.Name == "Hibernating Rhinos");
	 
	// Numeric property comparison
	results = session.Query<Company>()
	    .Where(x => x.NumberOfHappyCustomers > 100);
	 
	// Filtering based on a nested property
	results = session.Query<Company>()
	    .Where(x => x.Employees.Count > 10);

All throughout the documentation we are going to use both flavors interchangeably, and we refer to both as Linq.

One more thing to note before we go any further - all queries shown above, except from the first one, perform no actual querying even though they are well defined. The reason for this is that the Linq provider, by nature, does not perform any querying unless it is forced to. The call to `ToArray()` in the first snippet did just that, and by that triggered the execution of the query.

#### More filtering options

Other than the `Where` clause, there are several other useful operators you could use to filter results.

`Any` can be used on collections of objects (or primitive lists) in your entities to return only those who satisfies a condition. RavenDB also supports an `In` operator, to make reverse `Any` comparisons easier:

	// Return only companies having at least one employee named "Ayende"
	IQueryable<Company> companies = from c in session.Query<Company>()
	                                where c.Employees.Any(employee => employee.Name == "Ayende")
	                                select c;
	 
	// Query on nested collections - will return any company with at least one developer
	// whose specialty is in C#
	companies = from c in session.Query<Company>()
	            where c.Employees.Any(x => x.Specialties.Any(sp => sp == "C#"))
	            select c;
	 
	// Using the In operator - return entities whose a field value is in a provided list
	companies = from c in session.Query<Company>()
	            where c.Country.In(new[] { "Israel", "USA" })
	            select c;

#### Projections

Projections are specific fields projected from documents using the Linq `Select` method, they are not the original object but a new object that is being created on the fly and populated with results gathered by the query.

RavenDB Linq queries support projections, but its important to know that projected entities are not being tracked for changes. That is true even if the projected type is a named type (and not just an anonymous type).

Here is how to use projections:

	// In this sample, we are only interested in the names of the companies satisfying
	// our query conditions, so we project those only into an anonymous object.
	var companyNames = from c in session.Query<Company>()
	                   where c.Employees.Any(x => x.Specialties.Any(sp => sp == "C#"))
	                   select new { c.Name }; // This is where the projection happens
	 
	// Same query same idea, but this time we want to get results as objects of type Company.
	// Only the Name property will be populated, the rest will remain empty.
	Company[] companies = (from c in session.Query<Company>()
	                       where c.Employees.Any(x => x.Specialties.Any(sp => sp == "C#"))
	                       select new Company { Name = c.Name }) // This is where the projection happens
	    .ToArray();

Projections are useful when only part of the data is needed for your operation. Whenever change tracking isn't required, you're advised to consider using projections to ease bandwidth traffic between you and the server. This isn't a general rule, because caching in the entire application also plays an important role here, and it might make it more efficient to load the cache results of a query than to issue a remote query for a projection.
You can also use the `Distinct` method to only return distinct results from the server. When using projections, that means that on the server side, the database will compare all the projected fields for each object, and send us only unique results. If you aren't using projections, this has no effect but causing the server to do more work.

#### Sorting

You can use the `orderby` / `.OrderBy()` / `.OrderByDescending()` clauses to perform sorting. As we will see in a later chapter, more advanced sorting options are supported using static indexes.

#### Aggregate operators

RavenDB only supports the `Count` and `Distinct` Linq aggregate operators. For more complex aggregations, you'll need to make use of Map/Reduce indexes.

Similarly, the `SelectMany`, `GroupBy` and `Join` operators are not supported. Such operations should be made through a Map/Reduce index, and not while querying.

The `let` keyword is not supported currently.
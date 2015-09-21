---
layout: post
date: 2015-09-21
title: Dropwizard JDBI DAO trips & tests setup 
---

JDBI List Binding
=================
Today I encoutered a very good problem in dropwizard JDBI setup - how to bind a list to a DBI command?

Update your gradle file with this:

{% highlight groovy %}
    compile 'org.antlr:stringtemplate:3.2'
{% endhighlight %}

Let's say this is your DAO class. The code below is what you have in dropwizard documentation.

{% highlight java %}
@RegisterMapper(HotelMapper.class)
public interface HotelsDAO {
    @SqlQuery("SELECT id FROM hotels WHERE location_id=:location_id")
    public List<Integer> getHotelIdsForLocation(@Bind("location_id") int id);
}
{% endhighlight %}

However, the docs doesn't refer to how bind a list of stuff into the query. After a lot of searching, I found the solution

{% highlight java %}
import java.util.List;

import org.skife.jdbi.v2.sqlobject.Bind;
import org.skife.jdbi.v2.sqlobject.SqlQuery;
import org.skife.jdbi.v2.sqlobject.customizers.Define;
import org.skife.jdbi.v2.sqlobject.stringtemplate.UseStringTemplate3StatementLocator;

@UseStringTemplate3StatementLocator //(1)
public interface HotelsDAO {
    @SqlQuery("SELECT id FROM provider_hotels WHERE hotel_id in (<hotel_ids>)") //(2)
    public List<Integer> getWegoHotelIdsForProviderHotelIds(
            @Define("hotel_ids") List<Integer> hotelIds); //(3)

    @RegisterMapper(HotelMapper.class) //(4)
    @SqlQuery("SELECT id FROM hotels WHERE location_id=:location_id")
    public List<Integer> getHotelIdsForLocation(@Bind("location_id") int id);
}
{% endhighlight %}

The trick here is to _not_ use the param binding and substitute it with _string interpolation_. 

- (1) is there to tell JDBI that you want to use string interpolation using string template 3. Turns out that only version 3.x works

- (2) notices the `<hotel_ids>` instead of the usual `:hotels` in normal binding. Remember we are not doing an argument binding here - this is a string interpolation

- (3) here we use `Define` instead of `Bind` like normal. Same reason as in (2)

- (4) is not related to above points but still important nevertherless. In dropwizard docs I didn't know what one can register the mapping per method like this

Set up for DAO testing
======================

To set up test for dropwizard DAO, add these to your gradle file and substitute your version of dropwizard & h2db into `dropwizardVersion` and `h2DbVersion` respectively, or define them in `gradle.properties` file.

{% highlight groovy %}
testCompile "io.dropwizard:dropwizard-testing:$dropwizardVersion"   //dropwizardVersion=0.8.0
testCompile "com.h2database:h2:$h2DbVersion"    //h2DbVersion=1.4.189
{% endhighlight %}

Before this project I didn't know JUnit support convenient classes called `Rule` to group setting up/tearing down codes into one place. The way dropwizard do it is actually from junit. What you want do here is before the test suite, you set up the database and tear it down when the test suit ends. Here's the rule

{% highlight java %}
import org.junit.rules.ExternalResource;


public class H2JDBIRule extends ExternalResource {
    private DBI dbi;
    private Handle handle;

    @Override
    protected void before() throws Throwable {
        Environment environment = new Environment("test-env",
                Jackson.newObjectMapper(), null, new MetricRegistry(), null);
        dbi = new DBIFactory().build(environment, getDataSourceFactory(),
                "test");
        handle = dbi.open();
        createDatabase();
    }

    /**
    * This is where you create your databases using handle.createScript() or handle.createStatement() and so on....Remember to 
    * wrap them with handle.begin() and handle.commit() so that the change is visible for test code
    */
    public void createDatabase() {
        handle.begin();
        handle.createScript(FixtureHelpers.fixture("db/structure.sql"))
                .execute();
        //and so on
        handle.commit();
    }

    /**
    * I'm using h2db so I don't bother clearing up the database...You got the idea
    */
    @Override
    protected void after() {
        handle.close();
    }

    private DataSourceFactory getDataSourceFactory() {
        DataSourceFactory dataSourceFactory = new DataSourceFactory();
        dataSourceFactory.setDriverClass("org.h2.Driver");
        dataSourceFactory.setUrl(String.format(
                 "jdbc:h2:mem:test-%s;MODE=MySQL;TRACE_LEVEL_FILE=3",   //this is for in-memory db (*)
//              "jdbc:h2:./db/test-%s;MODE=MySQL;TRACE_LEVEL_FILE=3",   //this is for file db
                System.currentTimeMillis()));
        dataSourceFactory.setUser("sa");
        return dataSourceFactory;
    }

}
{% endhighlight %}

Notice the bit at (2). `MODE=MySQL` is to specify what dialect you want to use, `TRACE_LEVEL_FILE=3` is to specifiy the level of verboseness for debugging.

The table creation script is copied into `structure.sql` file as seen above. As I setup the db, I notice that 

- the part at the end of the create table script like `ENGINE=InnoDB AUTO_INCREMENT=3897 DEFAULT CHARSET=latin1` often causes syntax error for h2db

- the part `CHARACTER SET latin1` or similar charset _always_ causes syntax error

- sometimes the key defining part can cause syntax error as well, so I just remove whatever causes that and retry

Time to use that rule in your test suite. What you want is to use it as `ClassRule` so that it gets started before the whole test suite and cleared out after everything is done. You could argue that it's better to start the whole set up before and after _each_ test, but for cases where the tables are foreigned key to each other, setting up could be pretty slow. 

{% highlight java %}

    @ClassRule
    public static H2JDBIRule rule = new H2JDBIRule();

    @Before
    public void setup(){
        rule.getDBI().start();
        //do your further db setup in here
        rule.getDBI().commit();
    }

    @Test
    public void testXYZ(){
        HotelsDAO hotelsConnector = rule.getDbi().open(HotelsDAO.class);
        //do stuff with your DAO here
    }

    @After
    public void tearDown(){
        rule.getDBI().start();
        //reverse your changes here
        rule.getDBI().commit();
    }


{% endhighlight %}

That's how you do it!

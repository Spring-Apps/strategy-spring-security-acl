# strategy-spring-security-acl [![Build Status](https://travis-ci.org/lordlothar99/strategy-spring-security-acl.svg?branch=master)](https://travis-ci.org/lordlothar99/strategy-spring-security-acl)

Extensible strategy-based Spring Security ACL implementation ; available modules are : PermissionEvaluator, JPA Specification and ElasticSearch Filter

How to install : have a look here : [Installation](#Installation)

## Why ??

### Default [Spring Security ACL][] implementation is database-oriented

[Spring Security ACL][] default implementation uses a persistent model in order to evaluate every permission for any object and SID. Aside performance issues due to a huge ternary table in this model (1 row is 3-tuple { sid ; object weak reference ; permission }), developpers would rather use a more programmatic and object-oriented implementation, without any duplication.

### Access Control List is not only a PermissionEvaluator concern

Dealing with Access Control List is not a PermissionEvaluator-only concern. Let's take a CRUD-like standard application :
- Create, Update and Delete methods should be annotated with `@PreAuthorize("hasPermission(<object>, <permission>)")`, and so would reject unauthorized executions, that's great !
- But Read methods would use `@PostAuthorize("hasPermission(<object>, <permission>)")` instead, and therefore try to filter objects after they've been retrieved from underlying layers (often database) ; this leads to two frequent issues:
- performance : some useless objects are retrieved but evicted
- pagination : let's say user ask for a 10 items page ; repository finds those, but 4 of them are evicted... either developper implemented some retry-til-page-is-complete behavior (ugly pattern, and downgraded performance expected...), either client will get only 6 objects, and therefore ask himself where the 4 others went !! too bad...

## Ok, what's the proposal then ??

Strategy-spring-security-acl relies on several principles :

### Easy to plug extension of [Spring Security][]

Propose an alternative to [Spring Security Acl][]

### Easy to configure

Thanks to [Spring Boot][] auto-configure awesome magic

### Extensibility !!

Current bundled features are:
* Grant : implementation of PermissionEvaluator which delegates to adequate `GrantEvaluator` beans
* JPA : injects `JpaSpecification` in your repositories so they would retrieve only authorized objects from database ; thanks to [Spring Data JPA][]
* ElasticSearch : injects `FilterBuilder` in your repositories so they would retrieve only authorized objects from ElasticSearch ; thanks to [Spring Data ElasticSearch][]
You need more than existing features ? Create your own !! and share it ;).

### Strategy oriented

1. Create ACL strategies as Spring beans. Strategies are composite objects wrapping ACL filter components (1 for each feature)  
2. Install for each business object, and you

## <a name="Installation">Installation</a>

Add Github as a maven repository (yes, you can) :

	<repositories>
		<repository>
			<id>strategy-spring-security-acl-github-repo</id>
			<url>https://raw.github.com/lordlothar99/strategy-spring-security-acl/mvn-repo/</url>
			<snapshots>
				<enabled>true</enabled>
				<updatePolicy>always</updatePolicy>
			</snapshots>
		</repository>
	</repositories>

### [Spring Boot][]

We love [Spring Boot][]. Configured beans are automatically loaded by [Spring Boot][]'s magic, as soon jars are in the path.
Add required dependencies to your pom :

	<dependency>
		<groupId>com.github.lothar.security.acl</groupId>
		<artifactId>strategy-spring-security-acl-elasticsearch</artifactId>
		<version>LATEST</version>
	</dependency>

	<dependency>
		<groupId>com.github.lothar.security.acl</groupId>
		<artifactId>strategy-spring-security-acl-grant</artifactId>
		<version>LATEST</version>
	</dependency>

	<dependency>
		<groupId>com.github.lothar.security.acl</groupId>
		<artifactId>strategy-spring-security-acl-jpa</artifactId>
		<version>LATEST</version>
	</dependency>

Then you need very few Spring config:
* For Jpa feature :
```
	import com.github.lothar.security.acl.jpa.repository.AclJpaRepositoryFactoryBean;
...
	@EnableJpaRepositories(
		value = "<your jpa repositories package here>",
		repositoryFactoryBeanClass = AclJpaRepositoryFactoryBean.class
	)
```

* For ElasticSearch feature :
```
	import com.github.lothar.security.acl.elasticsearch.repository.AclElasticsearchRepositoryFactoryBean;
...
	@EnableElasticsearchRepositories(
		value = "<your elastic search repositories package here>",
		repositoryFactoryBeanClass = AclElasticsearchRepositoryFactoryBean.class
	)
```

* For GrantEvaluator feature (if you want to enable Pre/Post annotations) :
```
	@EnableGlobalMethodSecurity(prePostEnabled = true)
```

Now, let's say you have a `Customer` domain entity in your project, and you need to restrict readable customers, so that only those whose last name is "Smith" can be retrieved.
1. Define your strategy : let's create an `CustomerAclStrategy`, which will contain our different ACL features implementations (1 implementation by feature). `SimpleAclStrategy` implementation is recommanded as a start. In your favorite `Configuration` bean, let's define :
```
  @Bean
  public SimpleAclStrategy customerStrategy() {
    return new SimpleAclStrategy();
  }
```
2. Create a `CustomerGrantEvaluator` bean, and install it inside the `CustomerStrategy`. Let's add a new bean into `Configuration` :
```
  @Bean
  public GrantEvaluator smithFamilyGrantEvaluator(CustomerRepository customerRepository,
      GrantEvaluatorFeature grantEvaluatorFeature) {
    GrantEvaluator smithFamilyGrantEvaluator = new CustomerGrantEvaluator(customerRepository);
    customerStrategy.install(grantEvaluatorFeature, smithFamilyGrantEvaluator);
    return smithFamilyGrantEvaluator;
  }
```
And create a dedicated `CustomerGrantEvaluator` class, it's close to Spring's `PermissionEvaluator` API :
```
import static com.github.lothar.security.acl.jpa.spec.AclJpaSpecifications.idEqualTo;
import org.springframework.security.core.Authentication;
import com.github.lothar.security.acl.sample.domain.Customer;
import com.github.lothar.security.acl.sample.jpa.CustomerRepository;

public class CustomerGrantEvaluator extends AbstractGrantEvaluator<Customer, String> {

  private CustomerRepository repository;

  public CustomerGrantEvaluator(CustomerRepository repository) {
    super();
    this.repository = repository;
  }

  @Override
  public boolean isGranted(Permission permission, Authentication authentication,
      Customer domainObject) {
    return "Smith".equals(domainObject.getLastName());
  }

  @Override
  public boolean isGranted(Permission permission, Authentication authentication, String targetId,
      Class<? extends Customer> targetType) {
    // thanks to JpaSpecFeature, repository will count only authorized customers !
    return repository.count(idEqualTo(targetId)) == 1;
  }
}
```
3. Add Pre/Post annotations on adequate methods :
```
  @PreAuthorize("hasPermission(#customer, 'SAVE')")
...
  @PreAuthorize("hasPermission(#customerId, 'com.github.lothar.security.acl.sample.domain.Customer', 'READ')")
```

### Struggling with integration ?

Have a look at our samples !!

[Spring Boot]: http://projects.spring.io/spring-boot/
[Spring Data JPA]: http://projects.spring.io/spring-data-jpa/
[Spring Data ElasticSearch]: http://projects.spring.io/spring-data-elasticsearch/
[Spring Security]: http://projects.spring.io/spring-security/
[Spring Security ACL]: https://github.com/spring-projects/spring-security/tree/master/acl

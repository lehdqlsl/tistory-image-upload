[TOC]

## 1. 다중 DB 설정 시, 알아야 할 것

1.  다중 DB는 Spring boot 처럼 Auto Configuration되지 않음

    - 설정파일(application.yml 또는 application.properties) 값을 읽어와서 연동 할 DB 수 만큼 Datasource를 수동 설정해야함

    - 주요 설정 내용

      - 경로 설정

        - Repository basePackages 경로 설정
        - Entity 경로 설정

      - Datasource 설정

        - driver 이름
        - URL
        - Id/Password

      - Hibernate 설정

        - ddl-auto

        - dialect

          

2. 각각의 DB는 Repository package로 구분

3. 초기 설정이 복잡한 편이나, 천천히 살펴보면 크게 어렵지 않음

   

## 2. 소스코드

이 글에서는 1개의 Entity로 2개의 Database에 값을 넣는 예제로 진행합니다.

### 2-1. Entity

```java
package com.jpa.master.entity;

@Entity
@Table(schema = "users")
public class User {
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private int id;

	private String name;

	@Column(unique = true, nullable = false)
	private String email;

	private int age;
}
```



### 2-2. Repository

서로 다른 패키지에 Repository 작성, **패키지가 다르다고 하여 파일명을 동일하게 하면 안됨**

```java
package com.jpa.master.repository;

@Repository
public interface UserMasterRepository extends JpaRepository<User, Integer> {}
```

```java
package com.jpa.second.repository;

@Repository
public interface UserSecondRepository extends JpaRepository<User, Integer> {}
```



### 2-3. Configuration

#### 2-3-1 application.properties

- Prefix부분인 spring.datasource, spring.second-datasource는 자유롭게 입력 가능
- 그 뒤에 property명은 오타가 나지 않도록 주의 **(username, password, jdbcUrl)**

```properties
#### Main DataSource Configuration1
spring.datasource.username=[user]
spring.datasource.password=[Password]
spring.datasource.jdbcUrl=[URL]
#### Second DataSource Configuration2
spring.second-datasource.username=[user]
spring.second-datasource.password=[Password]
spring.second-datasource.jdbcUrl=[URL]
```

> driver class name을 설정하는 property가 빠져있는데,
>
> jdbcUrl로 부터 driver name을 읽어와서 자동으로 설정하기 때문에 별도로 설정 하지 않아도 됨



#### 2-3-2 Main Datasource

- 2개의 Config 설정 파일 생성

- 각 설정 파일은 다음과 같이 구성

  - DataSource 설정

  - EntityManagerFactory 설정

  - TransactionManager 설정

    

> 다중 DB를 설정할 때는, @Primary 어노테이션을 이용하여 Main이되는 Datasouce를 지정해야한다. 



```java
@Configuration
@PropertySource({ "classpath:application.properties" })
@EnableJpaRepositories(
    basePackages = "com.jpa.master.repository", // Repository 경로
    entityManagerFactoryRef = "masterEntityManager", 
    transactionManagerRef = "masterTransactionManager"
)
public class MainDatabaseConfig {
	@Autowired
	private Environment env;

	@Bean
	@Primary
	public LocalContainerEntityManagerFactoryBean masterEntityManager() {
		LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
		em.setDataSource(masterDataSource());
        
		//Entity 패키지 경로
        em.setPackagesToScan(new String[] { "com.jpa.master.entity" });
	
		HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
		em.setJpaVendorAdapter(vendorAdapter);
        
        //Hibernate 설정
		HashMap<String, Object> properties = new HashMap<>();
		properties.put("hibernate.hbm2ddl.auto"
                       ,env.getProperty("hibernate.hbm2ddl.auto"));
		properties.put("hibernate.dialect"
                       ,env.getProperty("hibernate.dialect"));
		em.setJpaPropertyMap(properties);
		return em;
	}
	
	@Primary
	@Bean
	@ConfigurationProperties(prefix="spring.datasource")
	public DataSource masterDataSource() {
		// @ConfigurationProperties을 사용하지 않을 경우 아래와 같이 작성
        // property명은 자유롭게 설정 가능
//		DriverManagerDataSource dataSource = new DriverManagerDataSource();
//		dataSource.setDriverClassName(env.getProperty("jdbc.driverClassName"));
//		dataSource.setUrl(env.getProperty("jdbc.main.url"));
//		dataSource.setUsername(env.getProperty("jdbc.user"));
//		dataSource.setPassword(env.getProperty("jdbc.password"));
//		return dataSource;
		
        //@ConfigurationProperties 어노테이션과 DataSourceBuilder를 이용하여 간단하게 설정 가능
		return DataSourceBuilder.create().build();
	}
	
	@Primary
	@Bean
	public PlatformTransactionManager masterTransactionManager() {	
		JpaTransactionManager transactionManager = new JpaTransactionManager();
		transactionManager.setEntityManagerFactory(masterEntityManager().getObject());
		return transactionManager;
	}
}
```



#### 2-3-3 Second Datasource

- **소스코드 복붙하다가 하는 실수**

  - @Primary 어노테이션은 꼭 지워야 함

```java
@Configuration
@PropertySource({ "classpath:application.properties" })
@EnableJpaRepositories(
    basePackages = "com.jpa.second.repository", 
    entityManagerFactoryRef = "secondEntityManager", 
    transactionManagerRef = "secondTransactionManager"
)
public class SecondConfig {
	@Autowired
	private Environment env;

	@Bean
	public LocalContainerEntityManagerFactoryBean secondEntityManager() {
		LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
		em.setDataSource(secondDataSource());
		em.setPackagesToScan(new String[] { "com.jpa.master.entity" });

		HibernateJpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
		em.setJpaVendorAdapter(vendorAdapter);
		HashMap<String, Object> properties = new HashMap<>();
		properties.put("hibernate.hbm2ddl.auto"
                       , env.getProperty("hibernate.hbm2ddl.auto"));
		properties.put("hibernate.dialect"
                       , env.getProperty("hibernate.dialect"));
		em.setJpaPropertyMap(properties);
		return em;
	}

	@Bean
	@ConfigurationProperties(prefix="spring.second-datasource")
	public DataSource secondDataSource() {
		return DataSourceBuilder.create().build();
	}

	@Bean
	public PlatformTransactionManager secondTransactionManager() {

		JpaTransactionManager transactionManager = new JpaTransactionManager();
		transactionManager.setEntityManagerFactory(secondEntityManager().getObject());
		return transactionManager;
	}
}
```



## 4. Test

- 간단한 Insert Test 코드 작성

```java
@RunWith(SpringJUnit4ClassRunner.class)
@AutoConfigureTestDatabase(replace = Replace.NONE)
@SpringBootTest
public class JPAMultipleDBTest {
	@Autowired
	UserMasterRepository masterRepo;
	
	@Autowired
	UserSecondRepository secondRepo;

	@Test
	public void insertTest() {
		User user = new User();
		user.setAge(28);
		user.setEmail("test@spring.com");
		user.setName("name");
		
		secondRepo.save(user);
		masterRepo.save(user);
	}
}
```

### 4.1 hibernate 로그 확인

![image](https://user-images.githubusercontent.com/7637916/79104222-1ac0a800-7da9-11ea-9084-3f2fcd2ab497.png)



### 4.2 DB 데이터 확인

- 테이블이 각 DB에 생성되었음

![image](https://user-images.githubusercontent.com/7637916/79104690-092bd000-7daa-11ea-8232-843160b0850f.png)



- 각 DB 데이터 확인
  ![image](https://user-images.githubusercontent.com/7637916/79104756-2d87ac80-7daa-11ea-8033-507079554b0d.png)
  ![image](https://user-images.githubusercontent.com/7637916/79104838-5c9e1e00-7daa-11ea-9417-48e739ccece9.png)
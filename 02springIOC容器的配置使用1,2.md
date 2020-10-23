[TOC]

# maven的介绍与使用

### 1、maven的简单介绍

​		Maven是Apache下的项目管理工具，它由纯Java语言开发，可以帮助我们更方便的管理和构建Java项目。

​		maven的优点

​		1、  jar包管理：

​			a)   从Maven中央仓库获取标准的规范的jar包以及相关依赖的jar包，避免自己下载到错误的jar包；

​			b)   本地仓库统一管理jar包，使jar包与项目分离，减轻项目体积。

​		2、  maven是跨平台的可以在window、linux上使用。

​		3、  清晰的项目结构；

​		4、  多工程开发，将模块拆分成若干工程，利于团队协作开发。

​		5、  一键构建项目：使用命令可以对项目进行一键构建。

### 2、maven的安装

​	maven官网：https://maven.apache.org/

​	maven仓库：https://mvnrepository.com/

​	安装步骤：

```
1、安装jdk
2、从官网中下载对应的版本
3、解压安装，然后配置环境变量，需要配置MAVEN_HOME,并且将bin目录添加到path路径下
4、在命令行中输入mvn -v,看到版本信息表示安装成功
```

### 3、maven的基本常识

**maven如何获取jar包**

​		**maven通过坐标的方式来获取 jar包**，坐标组成为：公司/组织（groupId）+项目名（artifactId）+版本（version）组成，可以从互联网，本地等多种仓库源获取jar包

**maven仓库的分类**

​		本地仓库：本地仓库就是开发者本地已经下载下来的或者自己打包所有jar包的依赖仓库，本地仓库路径配置在maven对应的conf/settings.xml配置文件。

​		私有仓库：私有仓库可以理解为自己公司的仓库，也叫Nexus私服

​		中央仓库：中央仓库即maven默认下载的仓库地址，是maven维护的

**maven的常用仓库**

​		由于网络访问的原因，在国内如果需要下载国外jar包的时候会受限，因此一般在使用过程中需要修改maven的配置文件，将下载jar包的仓库地址修改为国内的源，常用的是阿里云的mvn仓库，修改配置如下：

```xml
<mirror>
<id>alimaven</id>
<name>aliyun maven</name>
<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
<mirrorOf>central</mirrorOf>
</mirror>
```

### 4、maven常用命令

- clean：清理编译后的目录
- compile：编译，只编译main目录，不编译test中的代码
- test-compile：编译test目录下的代码
- test：运行test中的代码
- package：打包，将项目打包成jar包或者war包
- install：发布项目到本地仓库，用在打jar包上，打成的jar包可以被其他项目使用
- deploy：打包后将其安装到pom文件中配置的远程仓库
- site：生成站点目录


---


# **(2)使用maven的方式来构建项目**

- **创建maven项目**

  定义项目的groupId、artifactId
  
  项目名：spring_study2

- **添加对应的pom依赖**

  pom.xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
      <modelVersion>4.0.0</modelVersion>
  
      <groupId>com.mashibing</groupId>
      <artifactId>spring_demo</artifactId>
      <version>1.0-SNAPSHOT</version>
  
      <dependencies>
          <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
          <dependency>
              <groupId>org.springframework</groupId>
              <artifactId>spring-context</artifactId>
              <version>5.2.3.RELEASE</version>
          </dependency>
      </dependencies>
  </project>
  ```

- **编写代码**

  Person.java

  ```java
  package com.mashibing.bean;
  public class Person {
      private int id;
      private String name;
      private int age;
      private String gender;
  
      public int getId() {
          return id;
      }
  
      public void setId(int id) {
          this.id = id;
      }
  
      public String getName() {
          return name;
      }
  
      public void setName(String name) {
          this.name = name;
      }
  
      public int getAge() {
          return age;
      }
  
      public void setAge(int age) {
          this.age = age;
      }
  
      public String getGender() {
          return gender;
      }
  
      public void setGender(String gender) {
          this.gender = gender;
      }
  
      @Override
      public String toString() {
          return "Person{" +
                  "id=" + id +
                  ", name='" + name + '\'' +
                  ", age=" + age +
                  ", gender='" + gender + '\'' +
                  '}';
      }
  }
  ```

- **测试**

  MyTest.java

```java
import com.mashibing.bean.Person;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("ioc.xml");
        Person person = (Person) context.getBean("person");
        System.out.println(person);
    }
}
```

## **总结：**

​		以上两种方式创建spring的项目都是可以的，但是在现在的企业开发环境中使用更多的还是maven这样的方式，无须自己处理jar之间的依赖关系，也无须提前下载jar包，只需要配置相关的pom即可，因此推荐大家使用maven的方式，具体的maven操作大家可以看maven的详细操作文档。

​		**搭建spring项目需要注意的点：**

​		1、一定要将配置文件添加到类路径中，使用idea创建项目的时候要放在resource目录下【**因为是从src根目录进行查找，如果找不到配置文件，会到target/classes中去查找配置文件；把xml配置文件放到resources中，会将xml文件编译到target/classes目录下**】

​		2、导包的时候别忘了commons-logging-1.2.jar包【maven方式不需要，因为会自动导入该包】

​		**细节点：**

​		1、**ApplicationContext就是IOC容器的接口**，可以通过此对象获取容器中创建的对象

==为什么项目中不常使用FileSystemXmlApplicationContext==？因为会产生一个问题：windows有C/D/E盘符的，而部署到linux系统是没有盘符的概念的，如果想部署的话，还要改变当前项目的路径，这样就比较麻烦了

**BeanFactory是所有spring代码中最上层的接口**，ApplicationContext也是BeanFactory的一个实现

​		2、==对象在Spring容器中默认是在容器创建完成的时候就已经创建完成，不是需要用的时候才创建，此种情况满足的是单例模式==

​		3、对象在IOC容器中存储的时候都是单例的，如果需要多例需要修改属性【scope=prototype；如何保证单例-》多级缓存池-》底层数据结构是Map】

​		4、**创建对象给属性赋值的时候是通过setter方法实现的**

​		5、**对象的属性是由setter/getter方法决定的，而不是定义的成员属性**【setId,setIId会导致bean中的id属性注入不进去】

### 2、spring对象的获取及属性赋值方式

##### 		**1、通过bean的id获取IOC容器中的对象（上面已经用过，并且id不能重复）**

##### 		**2、通过bean的类型获取对象（注意：当通过类型进行获取的时候，如果存在两个相同类型对象，将无法完成获取工作）**

​		MyTest.java

```java
import com.mashibing.bean.Person;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("ioc.xml");
        //根据bean标签的id来获取对象
        Person person = context.getBean("person",Person.class);
        
        //根据bean的类型来获取对象
        Person bean = context.getBean(Person.class);
        System.out.println(bean);
    }
}
```

注意：通过bean的类型在查找对象的时候，在配置文件中不能存在两个类型一致的bean对象，如果有的话，可以通过如下方法

MyTest.java

```java
import com.mashibing.bean.Person;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("ioc.xml");
        Person person = context.getBean("person", Person.class);
        System.out.println(person);
    }
}
```

##### 		**3、通过构造器给bean对象赋值**

==当需要从容器中获取对象的时候，最好要保留无参构造方法，因为底层的实现是反射==

Class clazz = Person.class;
Object obj = clazz.newInstance();//默认调用无参的构造方法，此方法已经被弃用

**Object obj=clazz.getDeclaredConstructor().newInstance();//现在反射采用这种方式生成对象**

**在进行框架配置的时候，可以使用xml文件，也可以使用注解的方式，很多同学觉得xml的方式比较麻烦，但是xml的配置还是要学习的，因为在项目开发过程中很多情况下是xml和注解一起工作的，而且xml配置的方式比较完整**




ioc.xml

```xml
    <!-- 根据属性值设置的时候，name的名称取决于set方法后面的参数首字母小写的名称 property-->
    <bean id="person" class="com.mashibing.bean.Person">
        <property name="id" value="1"></property>
        <property name="name" value="zhangsan"></property>
        <property name="age" value="18"></property>
        <property name="gender" value="男"></property>
        <property name="date" value="2020/10/20"></property>
    </bean>
    
	<!--给person类添加构造方法；使用构造器方法赋值的时候，参数的name属性是由什么来决定的？由构造方法的参数列表决定的-->
	
	<!--
	    name:参数列表的名称
	    value:实际的具体值
	    type:值的类型
	    index:值的下标，从0开始
	-->
	<bean id="person2" class="com.mashibing.bean.Person">
        <constructor-arg name="id" value="1"></constructor-arg>
        <constructor-arg name="name" value="lisi"></constructor-arg>
        <constructor-arg name="age" value="20"></constructor-arg>
        <constructor-arg name="gender" value="女"></constructor-arg>
    </bean>

	<!--在使用构造器赋值的时候可以省略name属性，但是此时就要求必须严格按照构造器参数的顺序来填写了-->
	<bean id="person3" class="com.mashibing.bean.Person">
        <constructor-arg value="1"></constructor-arg>
        <constructor-arg value="lisi"></constructor-arg>
        <constructor-arg value="20"></constructor-arg>
        <constructor-arg value="女"></constructor-arg>
    </bean>

	<!--如果想不按照顺序来添加参数值，那么可以搭配index属性来使用-->
    <bean id="person4" class="com.mashibing.bean.Person">
        <constructor-arg value="lisi" index="1"></constructor-arg>
        <constructor-arg value="1" index="0"></constructor-arg>
        <constructor-arg value="女" index="3"></constructor-arg>
        <constructor-arg value="20" index="2"></constructor-arg>
    </bean>
    
	<!--当有多个参数个数相同，不同类型的构造器的时候，可以通过type来强制类型-->
	<!-- 当有多个相同参数的构造方法存在的时候，默认情况下是覆盖的过程，后面的构造方法会覆盖前面的构造方法 -->
	将person的age类型设置为Integer类型
	public Person(int id, String name, Integer age) {
        this.id = id;
        this.name = name;
        this.age = age;
        System.out.println("Age");
    }

    public Person(int id, String name, String gender) {
        this.id = id;
        this.name = name;
        this.gender = gender;
        System.out.println("gender");
    }
    <!-- 如果非要赋值给另外一个构造方法的话，可以使用type的参数进行指定；spring中没有自动拆箱装箱的过程-->
	<bean id="person5" class="com.mashibing.bean.Person">
        <constructor-arg value="1"></constructor-arg>
        <constructor-arg value="lisi"></constructor-arg>
        <constructor-arg value="20" type="java.lang.Integer"></constructor-arg>
    </bean>
	<!--如果不修改为integer类型，那么需要type跟index组合使用-->
	 <bean id="person5" class="com.mashibing.bean.Person">
        <constructor-arg value="1"></constructor-arg>
        <constructor-arg value="lisi"></constructor-arg>
        <constructor-arg value="20" type="int" index="2"></constructor-arg>
    </bean>
```

==总结：在日常工作中，一般都是用name、value的方式，很少有人去使用index或type的方式，但是要注意各种情况出现的问题==

##### 		**4、通过命名空间为bean赋值，简化配置文件中属性声明的写法**

在官网文档中，搜xmlns:p

p命名空间，类似于JSP中的c(c:if,c:for)，struts（ognl）,thymeleaf(th)


​		1、导入命名空间

xmlns:p="http://www.springframework.org/schema/p"

头信息：xmlns(xml namespace) ,xsi(xml schema instance 定义了xml的规范)、xsi.schemaLocation(规范的地址)、xsd(xml schema definition)；

DTD(Document Type Definition文档类型定义)

在xml文件中使用的<bean>/<property>标签，在对应的文件中进行识别，然后才能解析这些标签，以高亮的形式展示出来

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

```

​		2、添加配置

使用p命名空间来给属性赋值

```xml
    <bean id="person6" class="com.mashibing.bean.Person" p:id="3" p:name="wangwu" p:age="22" p:gender="男"></bean>
```

##### 		**5、为复杂类型进行赋值操作**

​		在之前的测试代码中，我们都是给最基本的属性进行赋值操作，在正常的企业级开发中还会遇到给各种复杂类型赋值，如集合、数组、其他对象等。

​		Person.java

```java
package com.mashibing.bean;

import java.util.*;

public class Person {
    private int id;
    private String name="dahuang";
    private int age;
    private String gender;
    private Address address;
    private String[] hobbies;
    private List<Book> books;
    private Set<Integer> sets;
    private Map<String,Object> maps;
    private Properties properties;

    public Person(int id, String name, int age, String gender) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.gender = gender;
        System.out.println("有参构造器");
    }

    public Person(int id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
        System.out.println("Age");
    }

    public Person(int id, String name, String gender) {
        this.id = id;
        this.name = name;
        this.gender = gender;
        System.out.println("gender");
    }

    public Person() {
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }

    public List<Book> getBooks() {
        return books;
    }

    public void setBooks(List<Book> books) {
        this.books = books;
    }

    public Map<String, Object> getMaps() {
        return maps;
    }

    public void setMaps(Map<String, Object> maps) {
        this.maps = maps;
    }

    public Properties getProperties() {
        return properties;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    public String[] getHobbies() {
        return hobbies;
    }

    public void setHobbies(String[] hobbies) {
        this.hobbies = hobbies;
    }

    public Set<Integer> getSets() {
        return sets;
    }

    public void setSets(Set<Integer> sets) {
        this.sets = sets;
    }

    @Override
    public String toString() {
        return "Person{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                ", gender='" + gender + '\'' +
                ", address=" + address +
                ", hobbies=" + Arrays.toString(hobbies) +
                ", books=" + books +
                ", sets=" + sets +
                ", maps=" + maps +
                ", properties=" + properties +
                '}';
    }
}


```

Book.java

```java
package com.mashibing.bean;

public class Book {
    private String name;
    private String author;
    private double price;

    public Book() {
    }

    public Book(String name, String author, double price) {
        this.name = name;
        this.author = author;
        this.price = price;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAuthor() {
        return author;
    }

    public void setAuthor(String author) {
        this.author = author;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }

    @Override
    public String toString() {
        return "Book{" +
                "name='" + name + '\'' +
                ", author='" + author + '\'' +
                ", price=" + price +
                '}';
    }
}

```

Address.java

```java
package com.mashibing.bean;

public class Address {
    private String province;
    private String city;
    private String town;

    public Address() {
    }

    public Address(String province, String city, String town) {
        this.province = province;
        this.city = city;
        this.town = town;
    }

    public String getProvince() {
        return province;
    }

    public void setProvince(String province) {
        this.province = province;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public String getTown() {
        return town;
    }

    public void setTown(String town) {
        this.town = town;
    }

    @Override
    public String toString() {
        return "Address{" +
                "province='" + province + '\'' +
                ", city='" + city + '\'' +
                ", town='" + town + '\'' +
                '}';
    }
}
```

ioc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/util https://www.springframework.org/schema/util/spring-util.xsd"
>

    <!--给复杂类型的赋值都在property标签内进行-->
    <bean id="person" class="com.mashibing.bean.Person">
        <property name="name">
            <!--赋空值-->
            <null></null>
        </property>
        <!--通过ref引用其他对象，引用外部bean-->
        <property name="address" ref="address"></property>
        <!--引用内部bean-->
       <!-- <property name="address">
            <bean class="com.mashibing.bean.Address">
                <property name="province" value="北京"></property>
                <property name="city" value="北京"></property>
                <property name="town" value="西城区"></property>
            </bean>
        </property>-->
        <!--为list赋值-->
        <property name="books">
            <list>
                <!--内部bean，无法从IOC容器中直接获取对象的值-->
                <bean id="book1" class="com.mashibing.bean.Book">
                    <property name="name" value="多线程与高并发"></property>
                    <property name="author" value="马士兵"></property>
                    <property name="price" value="1000"></property>
                </bean>
                <!--外部bean，可以随意从IOC容器获取对象的值-->
                <ref bean="book2"></ref>
            </list>
        </property>
        <!--给map赋值-->
        <property name="maps" ref="myMap"></property>
        <!--给property赋值-->
        <property name="properties">
            <props>
                <prop key="aaa">aaa</prop>
                <prop key="bbb">222</prop>
            </props>
        </property>
        <!--给数组赋值-->
        <property name="hobbies">
            <array>
                <value>book</value>
                <value>movie</value>
                <value>game</value>
            </array>
        </property>
        <!--给set赋值，重复值不显示-->
        <property name="sets">
            <set>
                <value>111</value>
                <value>222</value>
                <value>222</value>
            </set>
        </property>
    </bean>
    <bean id="address" class="com.mashibing.bean.Address">
        <property name="province" value="河北"></property>
        <property name="city" value="邯郸"></property>
        <property name="town" value="武安"></property>
    </bean>
    <bean id="book2" class="com.mashibing.bean.Book">
        <property name="name" value="JVM"></property>
        <property name="author" value="马士兵"></property>
        <property name="price" value="1200"></property>
    </bean>
    <!--级联属性-->
    <bean id="person2" class="com.mashibing.bean.Person">
        <property name="address" ref="address"></property>
        <property name="address.province" value="北京"></property>
    </bean>
    <!--util名称空间创建集合类型的bean-->
    <util:map id="myMap">
            <entry key="key1" value="value1"></entry>
            <entry key="key2" value-ref="book2"></entry>
            <entry key="key03">
                <bean class="com.mashibing.bean.Book">
                    <property name="name" value="西游记" ></property>
                    <property name="author" value="吴承恩" ></property>
                    <property name="price" value="100" ></property>
                </bean>
            </entry>
            <entry>
                <key>
                    <value>hehe</value>
                </key>
                <value>haha</value>
            </entry>
            <entry key="list">
                <list>
                    <value>11</value>
                    <value>12</value>
                </list>
            </entry>
    </util:map>
</beans>
```

##### 6、继承关系bean的配置

ioc.xml

```xml
    <bean id="person" class="com.mashibing.bean.Person">
        <property name="id" value="1"></property>
        <property name="name" value="zhangsan"></property>
        <property name="age" value="21"></property>
        <property name="gender" value="男"></property>
    </bean>
    <!--parent:指定bean的配置信息继承于哪个bean-->
    <bean id="person2" class="com.mashibing.bean.Person" parent="person">
        <property name="name" value="lisi"></property>
    </bean>
```

**如果想实现Java文件的抽象类，不需要将当前bean实例化的话，可以使用abstract属性**

abstract属性定义抽象bean，无法进行实例化

```xml
 	<bean id="person" class="com.mashibing.bean.Person" abstract="true">
        <property name="id" value="1"></property>
        <property name="name" value="zhangsan"></property>
        <property name="age" value="21"></property>
        <property name="gender" value="男"></property>
    </bean>
    <!--parent:指定bean的配置信息继承于哪个bean-->
    <bean id="person2" class="com.mashibing.bean.Person" parent="person">
        <property name="name" value="lisi"></property>
    </bean>
```

##### 7、bean对象创建的依赖关系

​		bean对象在创建的时候是按照bean在配置文件的顺序决定的，也可以使用depend-on标签来决定顺序

当bean对象被创建的时候，是按照xml配置文件定义的顺序创建的，谁在前谁就先被创建，如果需要干扰创建的顺序可以使用depend-on属性

一般在实际工作中不必在意bean创建的顺序，无论谁先创建，需要依赖的对象在创建完成之后都会进行赋值操作【**因为对象中的属性都为空，所以不必在意谁先创建**】

ioc.xml

```xml
	<bean id="book" class="com.mashibing.bean.Book" depends-on="person,address"></bean>
    <bean id="address" class="com.mashibing.bean.Address"></bean>
    <bean id="person" class="com.mashibing.bean.Person"></bean>
```

##### 8、bean的作用域控制，是否是单例==========================

ioc.xml

```xml
    <!--
    bean的作用域：singleton、prototype、request、session
    默认情况下是单例的
    prototype：多例模式(对应设计模式的：原型模式)，从IOC容器中获取的对象每次都是新创建的
        多实例的
        容器启动的时候不会创建多实例bean，只有在获取对象的时候才会创建该对象
        每次创建都是一个新的对象
    singleton：单例模式，默认的单例对象
        从IOC容器中获取的都是同一个对象，默认的作用越
        在容器启动完成之前就已经创建好对象
        获取的所有对象都是同一个
        
    在spring4.x版本中还包含另外两个作用域：
        request：每次发送请求都会有一个新的对象
        session：每一次会话都会有一个新的对象
        几乎不用，从来没用过，因此在5.x版本的时候被淘汰了
        
        打开浏览器，上淘宝，每一个商品都是一个连接；而打开的浏览器就是一个会话，默认为30分钟
    
    注意：
        如果是singleton作用域的话，每次在创建IOC容器完成之前此对象已经创建完成
        如果是prototype作用域的话，每次是在需要用到此对象的时候才会创建
    
    -->
    <bean id="person4" class="com.mashibing.bean.Person" scope="prototype"></bean>
```

##### 9、利用工厂模式创建bean对象

==设计模式的书《设计模式之禅》==

​		在之前的案例中，所有bean对象的创建都是通过反射得到对应的bean实例，其实在spring中还包含另外一种创建bean实例的方式，就是通过工厂模式进行对象的创建

​		在利用工厂模式创建bean实例的时候有两种方式，分别是静态工厂和实例工厂。

​		静态工厂：工厂本身不需要创建对象，但是可以通过静态方法调用，对象=工厂类.静态工厂方法名（）；

​		实例工厂：工厂本身需要创建对象，工厂类 工厂对象=new 工厂类；工厂对象.get对象名（）；

PersonStaticFactory.java

```java
package com.mashibing.factory;

import com.mashibing.bean.Person;

public class PersonStaticFactory {

    public static Person getPerson(String name){
        Person person = new Person();
        person.setId(1);
        person.setName(name);
        return person;
    }
}
```

ioc.xml

```xml
<!--
静态工厂的使用：类名.静态方法
class:指定静态工厂类
factory-method:指定哪个方法是工厂方法
-->
<bean id="person5" class="com.mashibing.factory.PersonStaticFactory" factory-method="getPerson">
        <!--constructor-arg：可以为方法指定参数-->
        <constructor-arg value="lisi"></constructor-arg>
    </bean>
```

PersonInstanceFactory.java

```java
package com.mashibing.factory;

import com.mashibing.bean.Person;

public class PersonInstanceFactory {
    public Person getPerson(String name){
        Person person = new Person();
        person.setId(1);
        person.setName(name);
        return person;
    }
}

```

ioc.xml

```xml
    <!--实例工厂使用：先创建工厂实例，然后调用工厂实例的方法-->
    <!--创建实例工厂类-->
    <bean id="personInstanceFactory" class="com.mashibing.factory.PersonInstanceFactory"></bean>
    <!--
    factory-bean:指定使用哪个工厂实例
    factory-method:指定使用工厂实例哪个的方法
    -->
    <bean id="person6" class="com.mashibing.bean.Person" factory-bean="personInstanceFactory" factory-method="getPerson">
        <constructor-arg value="wangwu"></constructor-arg>
    </bean>
```

**工厂方法 VS 抽象工厂**

```
工厂方法是产品层面的

抽象工厂是产品族层面的，抽象工厂更容易扩展

在spring创建bean对象的时候，两个都可以

工厂方法和抽象工厂最大的区别：一个是两层的，一个是三层的，怎么理解？举例：有一个CarFactory，

工厂方式是：里面有N多个方法，可以是静态方法

抽象方法是：再多抽象一层，比如说再抽象一层：小轿车Factory、公交Factory；小轿车Factory下面有大众、奥迪、宝马的Factory

spring源码中常使用抽象工厂

```


##### 10、继承FactoryBean来创建对象

BeanFactory定义的是规范，有N多个中方法；

FactoryBean是用来获取唯一的对象的，里面的有是三个方法：getObect()、getObjectType()、isSingleton()


​		**FactoryBean是Spring规定的一个接口，当前接口的实现类，Spring都会将其作为一个工厂，但是在ioc容器启动的时候不会创建实例，只有在使用的时候才会创建对象**


通过FactoryBean实现对象创建自己控制，并将FactoryBean交由spring容器管理

FactoryBean是泛型的，作用是：用于指定创建的是哪个对象或哪一类对象

**此方式是spring创建bean方式的一种补充：用户可以按照需求创建对象，创建的对象交由spring IOC容器来进行管理，无论是否是单例，都是在用到的时候才会创建该对象，不用该对象不会创建**

==此种方法自己用的并不多，但是在源码中使用比较多 -》 查看FactoryBean的实现类有很多==


MyFactoryBean.java

```java
package com.mashibing.factory;

import com.mashibing.bean.Person;
import org.springframework.beans.factory.FactoryBean;

/**
 * 实现了FactoryBean接口的类是Spring中可以识别的工厂类，spring会自动调用工厂方法创建实例
 */
public class MyFactoryBean implements FactoryBean<Person> {

    /**
     * 工厂方法，返回需要创建的对象（返回获取的bean）
     * @return
     * @throws Exception
     */
    @Override
    public Person getObject() throws Exception {
        Person person = new Person();
        person.setName("maliu");
        return person;
    }

    /**
     * 返回创建对象的类型,spring会自动调用该方法返回对象的类型（获取返回的bean的类型）
     * @return
     */
    @Override
    public Class<?> getObjectType() {
        return Person.class;
    }

    /**
     * 创建的对象是否是单例对象（判断当前bean是否是单例的）
     * @return
     */
    @Override
    public boolean isSingleton() {
        return false;
    }
}
```

ioc.xml

```xml
<!-- 将FactoryBean交由Spring容器管理 -->
<bean id="myfactorybean" class="com.mashibing.factory.MyFactoryBean"></bean>
```

```
//Test
Person person= context.getBean("myfactoryBean",Person.class);
```

==黄老师的spring源码课，讲的很全面，是从框架层面出发的，是一个比较高的角度出发的==


##### 11、bean对象的初始化和销毁方法


Python语言中，创建对象的时候会调用__new__这个方法的，但是要注意这个方法只是为了创建具体对象，除了这个方法之外，还有一个__init__的方法，而init方法当中更多的是 对象中某些属性进行一些基础的初始化工作




​		在创建对象的时候，我们可以根据需要调用初始化和销毁的方法

Address.java

```java
package com.mashibing.bean;

public class Address {
    private String province;
    private String city;
    private String town;

    public Address() {
        System.out.println("address被创建了");
    }

    public Address(String province, String city, String town) {
        this.province = province;
        this.city = city;
        this.town = town;
    }

    public String getProvince() {
        return province;
    }

    public void setProvince(String province) {
        this.province = province;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public String getTown() {
        return town;
    }

    public void setTown(String town) {
        this.town = town;
    }

    public void init(){
        System.out.println("对象被初始化");
    }
    
    public void destory(){
        System.out.println("对象被销毁");
    }
    
    @Override
    public String toString() {
        return "Address{" +
                "province='" + province + '\'' +
                ", city='" + city + '\'' +
                ", town='" + town + '\'' +
                '}';
    }
}
```

ioc.xml

```xml
<!--bean生命周期表示bean的创建到销毁
        如果bean是单例，容器在启动的时候会创建好，关闭的时候会销毁创建的bean(初始化和销毁的方法都存在)
        如果bean是多礼，获取的时候创建对象，销毁的时候不会有任何的调用（初始化方法会调用，但是销毁的方法不会调用）
    
    spring容器在创建对象的时候可以指定具体的初始化和销毁方法
    init-method：在对象创建完成之后会调用初始化方法
    destory-method：在容器关闭的时候会调用销毁方法;
    
    init、destory是定义的普通方法，不是固定名称的
    -->
    <bean id="address" class="com.mashibing.bean.Address" init-method="init" destroy-method="destory" scope="prototype"></bean>
```

MyTest.java

```java
import com.mashibing.bean.Address;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new  ClassPathXmlApplicationContext("ioc2.xml");
        Address address = context.getBean("address", Address.class);
        System.out.println(address);
        //applicationContext没有close方法，需要使用具体的子类
        ((ClassPathXmlApplicationContext)context).close();

    }
}

//运行结果：
address被创建了
对象被初始化
Address{id=0,name='',.....}
对象被销毁
```

applicationContext没有close方法，需要使用具体的子类

**证明：查看ClassPathXmlApplicationContext的父类，发现可以找到父类为：ConfigurableApplicationContext,该类中有close方法；ConfigurableApplicationContext继承ApplicatonContext**

##### 12、配置bean对象初始化方法的前后处理方法--》提高扩展性

​		spring中包含一个BeanPostProcessor的接口，可以在bean的初始化方法的前后调用该方法，如果配置了初始化方法的前置和后置处理器，无论是否包含初始化方法，都会进行调用


如果想利用BeanPostProcessor对具体的类做控制，可以在前置和后置处理器中增加逻辑处理

springboot的源码中有很多PostProcessor，就是对对应的类进行增强
例如：在springboot源码：org.springframework.boot:spring-boot:2.2.0.RELEASE的spring.factories中有很多的PostProcessor，里面先留个印象，方便以后看源码的时候便于理解

MyBeanPostProcessor.java

```java
package com.mashibing.bean;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

public class MyBeanPostProcessor implements BeanPostProcessor {
    /**
     * 在初始化方法调用之前执行
     * @param bean  初始化的bean对象
     * @param beanName  xml配置文件中的bean的id属性
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof Person){
            //可以对具体的类做处理
        }else {
            //
        }
        System.out.println("postProcessBeforeInitialization:"+beanName+"调用初始化前置方法");
        return bean;
    }

    /**
     * 在初始化方法调用之后执行
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization:"+beanName+"调用初始化后缀方法");
        return bean;
    }
}
```

ioc.xml

```xml
<bean id="myBeanPostProcessor" class="com.mashibing.bean.MyBeanPostProcessor"></bean>
<bean id="address" class="com.mashibing.bean.Address" init-method="init" destroy-method="destory"></bean>
<bean id="person" class="com.mashibing.bean.Person"></bean>
```


```
//Test
ApplicationContext context = new  ClassPathXmlApplicationContext("ioc2.xml");
Address address = context.getBean("address", Address.class);
System.out.println(address);

运行结果：
address被创建
postProcessBeforeInitialization:address
address对象初始化完成
postProcessAfterInitialization:address
对象被初始化
Address{id=0,name='',.....}
对象被销毁

person被创建
postProcessBeforeInitialization:person
postProcessAfterInitialization:person

```

### 3、spring创建第三方bean对象

​		在Spring中，很多对象都是单实例的，在日常的开发中，我们经常需要使用某些外部的单实例对象，例如数据库连接池，下面我们来讲解下如何在spring中创建第三方bean实例。

​		1、导入数据库连接池的pom文件

```xml
<!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.21</version>
</dependency>
<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>
```

​		2、编写配置文件

ioc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- spring管理第三方bean -->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="username" value="root"></property>
        <property name="password" value="123456"></property>
        <property name="url" value="jdbc:mysql://localhost:3306/demo"></property>
        <property name="driverClassName" value="com.mysql.jdbc.Driver"></property>
    </bean>
</beans>
```

​		3、编写测试文件

MyTest.java

```java
import com.alibaba.druid.pool.DruidDataSource;
import com.mashibing.bean.Address;
import com.mashibing.bean.Person;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import java.sql.SQLException;

public class MyTest {
    public static void main(String[] args) throws SQLException {
        ApplicationContext context = new ClassPathXmlApplicationContext("ioc3.xml");
        DruidDataSource dataSource = context.getBean("dataSource", DruidDataSource.class);
        System.out.println(dataSource);
        System.out.println(dataSource.getConnection());
    }
}
```

### 4、spring引用外部配置文件

在resource中添加dbconfig.properties

```properties
username=root
password=123456
url=jdbc:mysql://localhost:3306/demo
driverClassName=com.mysql.jdbc.Driver
```

编写配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 需要添加xmlns:context以及在xsi:schemaLocation中增加context（无法不配置这两行代码，无法使用；如果记不住，官网中相关的文档）

记忆方法：先添加命名空间xmlns:context;
再在shemalocation中添加context，把spring-beans的链接复制下来，将beans替换为context

-->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">
	<!--加载外部配置文件
		在加载外部依赖文件的时候需要context命名空间（之前已经学过p命名空间）
		
		location：可以是URL，也开始是classpath的伪路径；也可以是相对路径；如果有多个用逗号分隔，点击location可以查看文档注释，里面有
		
		值使用$表达式,$会从指定的配置文件中读取配置
		
		在配置文件编写属性的时候要注意：
		spring容器在进行启动的时候，会读取当前系统的某些环境变量的配置，当前系统的用户名的是用username来表示的，所以最好的方式是添加前缀来区分
		
		如果属性重名怎么办？加前缀：jdbc.username=limiao
	-->
    <context:property-placeholder location="classpath:dbconfig.properties"/>
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="username" value="${username}"></property>
        <property name="password" value="${password}"></property>
        <property name="url" value="${url}"></property>
        <property name="driverClassName" value="${driverClassName}"></property>
    </bean>
    
    <!-- 普通对象使用配置文件
        当前情况会读取到操作系统的系统变量，使用username的要小心，如果想避免这种问题，可以加前缀：jdbc.username=limiao
        value=${jdbc.username}
        如果属性重名怎么办？加前缀：jdbc.username=limiao
    -->
    <bean id="person" class="com.mashibing.bean.Person">
        <property name="name" value="${username}"></property>
    </bean>
</beans>
```

### 5、spring基于xml文件的自动装配

​		当一个对象中需要引用另外一个对象的时候，在之前的配置中我们都是通过property标签来进行手动配置的，其实在spring中还提供了一个非常强大的功能就是自动装配，可以按照我们指定的规则进行配置，配置的方式有以下几种：

​		default/no：不自动装配

​		byName：按照名字进行装配，根据set方法后面的名称首字母小写决定的，不是参数列表的名称；以属性名作为id去容器中查找组件，进行赋值，如果找不到则装配null【最常用】

​		byType：按照bean的类型进行装配,以属性的类型作为查找依据去容器中找到这个组件，如果有多个类型相同的bean对象，那么会报异常，如果找不到则装配null【打印Person类的时候，会多一个属性environment;是因为spring容器在默认情况下加载当前系统的环境变量的；具体可以看源码】

**为什么使用byType的时候会加载环境变量出来？** -》在context中有一个属性environment，是map结构，因为Person中的maps属性是Map<String,Object>，会把environment自动注册到Person的maps中；如果把maps的类型改成Map<String,String>就不会打印系统环境变量了



​		constructor：按照构造器进行装配，先按照有参构造器参数的类型进行装配，没有就直接装配null；**如果按照类型找到了多个bean，那么就使用参数名作为id继续匹配，找到就装配，找不到就装配null**

ioc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="address" class="com.mashibing.bean.Address">
        <property name="province" value="河北"></property>
        <property name="city" value="邯郸"></property>
        <property name="town" value="武安"></property>
    </bean>
    <!-- 在spring中，可以使用自动装配的功能，spring会把某些bean注入到另外bean中，可以使用autowire属性来实现自动装配，有以下几种情况：
        default/no:
        byName:
        byType:
        constructor:
        
    -->
    <bean id="person" class="com.mashibing.bean.Person" autowire="byName"></bean>
    <bean id="person2" class="com.mashibing.bean.Person" autowire="byType"></bean>
    <bean id="person3" class="com.mashibing.bean.Person" autowire="constructor"></bean>
</beans>
```

### 6、SpEL的使用

在官网中有单独的模块来介绍SpEL

类似：jsp中的EL或jstl表达式  或者 struts中ognl表达式



​		SpEL:Spring Expression Language,spring的表达式语言，支持运行时查询操作对象

​		使用#{...}作为语法规则，所有的大括号中的字符都认为是SpEL.【外部配置文件，使用${}】

ioc.xml

```xml
    <!--SpEL表达式语言的使用-->
    <bean id="person4" class="com.mashibing.bean.Person">
        <!--支持运算符的所有操作-->
        <property name="age" value="#{12*2}"></property>
        <!--可以引用其他bean的某个属性值-->
        <property name="name" value="#{address.province}"></property>
        <!--引用其他bean-->
        <property name="address" value="#{address}"></property>
        <!--调用静态方法-->
        <property name="hobbies" value="#{T(java.util.UUID).randomUUID().toString().substring(0,4)}"></property>
        <!--调用非静态方法-->
        <property name="gender" value="#{address.getCity()}"></property>
    </bean>
```

# Spring项目，Spring八股，Spring Boot八股，在这一篇里面学会

**按照知识点进行学习，一个视频算是一个知识点。**

注意代码和知识点的结合

**@Bean 和下面的xml文件中，标签bean的作用是一样的，都是会被扫描，然后给beanFactory进行管理。**

另外在这里学习一下，@SpringBootApplication注解

`@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制

`@ComponentScan`：扫描被`@Component` (`@Repository`,`@Service`,`@Controller`)注解的 bean，注解默认会扫描该类所在的包下所有的类。

`@Configuration`：允许在 Spring 上下文中注册额外的 bean 或导入其他配置类



`@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制（自动装配）

https://www.cnblogs.com/Chary/articles/17721034.html（自动装配原理讲解）



------



# 1.初始bean相关的类

1. **BeanDefinition实现类**，顾名思义，用于定义bean信息的类，包含bean的class类型、构造参数、属性值等信息，每个bean对应一个BeanDefinition的实例。（beanDefinition用来描述bean,就像**class对象**用来描述一个类）

3. **BeanDefinitionRegistry接口**，实现了BeanDefinition注册表接口，定义注册BeanDefinition的方法。

4. **BeanFactory接口**，实现了基本的getbean,containsBean

5. **DefaultListableBeanFactory实现类**（也就是所说的bean容器！！！！！！！！）,实现了**BeanDefinitionRegistry接口**，

   能够注册beandefinition（把beanDefination放到beanDefinationMap里面），

   获得beandefiniton，

   获得bean。（重点是getBean方法，根据类名获得bean）

   

6. **第一次getbean时，会把单例对象存到一级缓存中，并删除二级三级缓存。（第二次getbean，可以直接从缓存中拿到）**

7. **DefaultSingletonBeanRegistry实现类，**单例bean注册类，实现了三级缓存，有获得单例对象（从缓存中get），还有向缓存中添加单例，以及销毁单例

8. **AbstractBeanFactory实现类**，有getBean方法，会从单例缓存中拿，没有的话就使用createBean创建bean,又会去使用**AbstractAutowireCapableBeanFactory的**docreatebean方法，然后根据beandefination创建bean

   

![image-20240618220943512](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240618220943512.png)

总结一下，

首先创建bean工厂

我们可以根据文件，或者自己传参，创建beanDefination,传递一些bean的参数

然后把这个bean注册到bean工厂

当我们需要使用bean实例时，使用bean工厂进行实例化，首先使用父类函数getbean，调用路径如下

AbstractBeanFactory           getbean

AbstractAutowireCapableBeanFactory      createBean

AbstractAutowireCapableBeanFactory	  doCreateBean（实例化，属性赋值，返回）



# 2.属性赋值

bean通过SimpleInstantiationStrategy实例化的话，最终走到这里，用 constructor.newInstance();进行实例化，

**（以上的流程）是根据beanDefinition的无参构造函数创建beanDefination,并在getbean时实例化对象**

注！实例化是在使用的时候，才实例化的！！！！

```java
public class SimpleInstantiationStrategy implements InstantiationStrategy {

	/**
	 * 简单的bean实例化策略，根据bean的无参构造函数实例化对象
	 *
	 * @param beanDefinition
	 * @return
	 * @throws BeansException
	 */
	@Override
	public Object instantiate(BeanDefinition beanDefinition) throws BeansException {
		Class beanClass = beanDefinition.getBeanClass();
		try {
			Constructor constructor = beanClass.getDeclaredConstructor();
			return constructor.newInstance();
		} catch (Exception e) {
			throw new BeansException("Failed to instantiate [" + beanClass.getName() + "]", e);
		}
	}
}
```

beanDefinition也可以通过传递属性参数，进行属性赋值。

流程同上，

先去查单例缓存，没有。

然后去获得beanDefination,

然后去creatBean ,   docreateBean

在AbstractAutowireCapableBeanFactory的docreateBean中，要先实例化一个bean，同上，就得到了一个属性值为空的bean。

然后在applyPropertyValues(beanName, bean, beanDefinition);中添加属性。**注，属性此时都保存在beanDefination里面**一个叫PropertyValues的类里面

然后一对一对的把属性拿出来，每个循环拿出一个属性名 ，一个属性value

然后拿出这个类里面的属性进行检查（类型检查，依赖检查）然后通过反射进行赋值

```
//通过反射设置属性
BeanUtil.setFieldValue(bean, name, value)
```

最终是setFieldValue(obj, field, value)设置了值

```java
public static void setFieldValue(Object obj, String fieldName, Object value) throws UtilException {
    Assert.notNull(obj);
    Assert.notBlank(fieldName);
    Field field = getField(obj instanceof Class ? (Class)obj : obj.getClass(), fieldName);
    Assert.notNull(field, "Field [{}] is not exist in [{}]", new Object[]{fieldName, obj.getClass().getName()});
    setFieldValue(obj, field, value);
}
```

经过这么一个流程，就给bean对象的属性成功赋值了，经过多个循环，就会都赋值。

![image-20240618224115643](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240618224115643.png)



这是没有类依赖的情况



**如果有类之间的依赖，如何属性赋值呢？**

这个car需要按照beanReference的方式传进去

![image-20240618225139793](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240618225139793.png)

于是在属性赋值循环的时候，会被发现是beanReference

![image-20240618225250280](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240618225250280.png)

然后会实例化这个beanReference，嵌套式的再实例化一个car类实例，把这个实例的值作为属性赋值给person类

**（总结就是，对于有类依赖的属性赋值，先把内层类实例化并赋值，再赋值给外层类！也就是，为bean注入bean）**





# 3.资源和资源加载器

三类资源，classpath资源（ClassPathResource）:  ，  url资源（UrlResource），   文件资源(FileSystemResource)

**这三类都是加载任意文件**

```
	if (location.startsWith(CLASSPATH_URL_PREFIX)) {
			//classpath下的资源
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()));
		} else {
			try {
				//尝试当成url来处理
				URL url = new URL(location);
				return new UrlResource(url);
			} catch (MalformedURLException ex) {
				//当成文件系统下的资源处理
				return new FileSystemResource(location);
			}
		}
```

资源是通过默认资源加载器（DefaultResourceLoader）加载的，这个资源加载器能够加载上面的三种资源，并且都是通过流进行加载的。先classpath，后尝试url，最后文件





**在XML文件中定义bean，注册到bean**

定义bean自不必多说，读取xml步骤：

创建bean工厂

创建实现类： XmlBeanDefinitionReader

使用 XmlBeanDefinitionReader 的  loadBeanDefinitions

loadBeanDefinitions先通过上面的资源加载器，进行资源加载。

再定义输入流，读取资源

```
Document document = reader.read(inputStream);
```

然后定义文件的根元素，像读取树一样，按照bean的标签进行读取。

```java
List<Element> beanList = root.elements(BEAN_ELEMENT);
		for (Element bean : beanList) {
			String beanId = bean.attributeValue(ID_ATTRIBUTE);
			String beanName = bean.attributeValue(NAME_ATTRIBUTE);
			String className = bean.attributeValue(CLASS_ATTRIBUTE);
			String initMethodName = bean.attributeValue(INIT_METHOD_ATTRIBUTE);
			String destroyMethodName = bean.attributeValue(DESTROY_METHOD_ATTRIBUTE);
			String beanScope = bean.attributeValue(SCOPE_ATTRIBUTE);
			String lazyInit = bean.attributeValue(LAZYINIT_ATTRIBUTE);
			Class<?> clazz;
```

然后对拿到类名，接着这些元素进行检查（判空）和处理（处理空值），如果id和name都为空，就取出clazz的第一个字符作为name

接着定义一个beanDefination（传进去一个类名），然后把刚刚从bean标签里面拿到的一些字段赋值给beanDefination（InitMethodName，DestroyMethodName，LazyInit）

然后开始读取property这个标签，也就是bean的属性。同样保存到list里面，然后赋值给beanDefination

```java
PropertyValue propertyValue = new PropertyValue(propertyNameAttribute, value);
beanDefinition.getPropertyValues().addPropertyValue(propertyValue);
```

**最后完成beanDefination的注册（把beanDefination放到beanDefinationMap里面）**





# 4.BeanFactoryPostProcessor和BeanPostProcessor

**BeanFactoryPostProcessor**是spring提供的容器扩展机制，**允许我们在bean实例化之前修改bean的定义信息即BeanDefinition的信息。**

（其重要的实现类有PropertyPlaceholderConfigurer和CustomEditorConfigurer，

PropertyPlaceholderConfigurer的作用是用properties文件的配置值替换xml文件中的占位符，

CustomEditorConfigurer的作用是实现类型转换。）





BeanFactoryPostProcessor是一个接口，没有任何内容

代码中的自定义实现类CustomBeanFactoryPostProcessor 实现了这个接口，内容也是自己定义的，可以在bean实例化之前修改beanDefinition，这里修改的是beanDefinition的propertyValues。

（例如，在这个CustomBeanFactoryPostProcessor里面，定义方法，给某个beanDefinition添加属性）







**BeanPostProcessor**也是spring提供的容器扩展机制，不同于BeanFactoryPostProcessor的是，**BeanPostProcessor在bean实例化后修改bean或替换bean。BeanPostProcessor是后面实现AOP的关键。**

**BeanPostProcessor接口定义了两个方法等待去实现：postProcessBeforeInitialization   和  postProcessAfterInitialization**

这里代码是自定义的BeanPostProcessor的两个方法，对已经实例化的bean做了处理，并且打印信息。

BeanPostProcessor会被添加到BeanPostProcessors(List)中，**这里面存放所有当前bean工厂的所有BeanPostProcessor**





BeanFactoryPostProcessor和BeanPostProcessor都是在xml文件中定义的，对于你想要的bean，需要特定的bean定义特定的xml文件

**（总结：xml文件中定义：bean的名称，属性，BeanFactoryPostProcessor和BeanPostProcesso，initmethod，destorymethod）**

![image-20240620160250985](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620160250985.png)

去debug,单步跟踪，在docreateBean方法中，会有一个步骤，对于那个bean，首先加载所有BeanPostProcessor，对于这些BeanPostProcessor，每一个都会使用前置处理方法处理bean。

然后调用初始化方法

然后同上，加载所有BeanPostProcessor，对于这些BeanPostProcessor，每一个都会使用后置处理方法处理bean。







# 5.应用上下文ApplicationContext

应用上下文ApplicationContext是spring中较之于BeanFactory更为先进的IOC容器，ApplicationContext除了**拥有BeanFactory的所有功能外**，还支持特殊类型bean如上一节中的BeanFactoryPostProcessor和BeanPostProcessor**的自动识别（注，我们之前在4中是手动调用FactoryPostProcessor和手动添加PostProcessor的）、资源加载、容器事件和监听器、国际化支持、单例bean自动初始化**等。

#### **BeanFactory是spring的基础设施，面向spring本身；而ApplicationContext面向spring的使用者，应用场合使用ApplicationContext**。

具体实现查看AbstractApplicationContext#refresh方法即可。



AbstractApplicationContext就是把我们之前手动的一系列操作相当于进行了打包

```java
public void refresh() throws BeansException {
		//创建BeanFactory，并加载BeanDefinition
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();

		//添加ApplicationContextAwareProcessor，让继承自ApplicationContextAware的bean能感知bean
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

		//在bean实例化之前，执行BeanFactoryPostProcessor
		invokeBeanFactoryPostProcessors(beanFactory);

		//BeanPostProcessor需要提前与其他bean实例化之前注册
		registerBeanPostProcessors(beanFactory);

		//初始化事件发布者
		initApplicationEventMulticaster();

		//注册事件监听器
		registerListeners();

		//注册类型转换器和提前实例化单例bean
		finishBeanFactoryInitialization(beanFactory);

		//发布容器刷新完成事件
		finishRefresh();
	}
```

注意看这几步，分别是

创建BeanFactory，加载BeanDefinition（根据xml）

![image-20240619173951344](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240619173951344.png)

获得BeanFactory

添加aware

（下面两个就是实现自动识别的关键）

**执行BeanFactoryPostProcessor**

```java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		Map<String, BeanFactoryPostProcessor> beanFactoryPostProcessorMap = beanFactory.getBeansOfType(BeanFactoryPostProcessor.class);
		for (BeanFactoryPostProcessor beanFactoryPostProcessor : beanFactoryPostProcessorMap.values()) {
			beanFactoryPostProcessor.postProcessBeanFactory(beanFactory);
		}
	}
```

注意，BeanFactoryPostProcessor，BeanPostProcessor都是bean！也被写到了xml文件里，读取到了beanDefinition里面

这里通过类名，就能拿到BeanFactoryPostProcessor（按照map存放）。然后遍历这个map，进行执行BeanFactoryPostProcessor的方法。

**注册BeanPostProcessor**

```java
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		Map<String, BeanPostProcessor> beanPostProcessorMap = beanFactory.getBeansOfType(BeanPostProcessor.class);
		for (BeanPostProcessor beanPostProcessor : beanPostProcessorMap.values()) {
			beanFactory.addBeanPostProcessor(beanPostProcessor);
		}
	}
```

同上，通过类名拿到BeanPostProcessor的map，然后遍历把这些BeanPostProcessor添加到beanFactory中。在后面单例实例化时，会自动使用前置和后置方法。（自动识别，通过自动把BeanPostProcessor添加到beanFactory中实现）









提前实例化单例bean，这里会把所有bean工厂里面的单例bean都实例化

![image-20240619174253864](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240619174253864.png)

![image-20240619174325848](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240619174325848.png)







# 6.bean的初始化方法和销毁方法

![image-20240620155918136](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620155918136.png)

（流程：实例化，初始化，使用，销毁）

bean定义初始化方法和销毁方法有三种方式（不冲突！可以同时存在，但是先后有顺序）

- 在xml文件里面定义了   initmethod   ,   destorymethod.
- 继承接口，然后自己实现

  



在docreateBean方法里的initializeBean里面（有前置处理，初始化，后置处理）

![image-20240620161325576](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620161325576.png)

具体的初始化方法是在invokeInitMethods(beanName, wrappedBean, beanDefinition)里面

![image-20240620161531233](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620161531233.png)

首先用

```
instanceof来判断是不是一个InitializingBean接口的实现类
```

如果实现了这个接口，就先使用接口的初始化方法。也就是第二种实现方式。



接着，获取beanDefination里面的初始化方法名，如果非空且不是第二种实现方式，就获得这个方法，

然后用反射的方式运行它。也就是第一种方式。









销毁方法类似，在docreateBean里面，会注册销毁方法

![image-20240620165412348](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620165412348.png)

如果是实现接口，或者bean定义里面存在，就注册（把bean放到map里面）

![image-20240620165559952](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620165559952.png)

具体的destory执行是在Adapter里面执行，顺序和执行方式同上，都是

“

首先用

```
instanceof来判断是不是一个  销毁ingBean接口的实现类
```

如果实现了这个接口，就先使用接口的初始化方法。也就是第二种实现方式。



接着，获取beanDefination里面的销毁方法名，如果非空且不是第二种实现方式，就获得这个方法，

然后用反射的方式运行它。也就是第一种方式。

”





# 7.BeanFactoryAware和ApplicationContextAware

![image-20240620173518171](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620173518171.png)

**BeanFactoryAware在实例化之后，前置处理之前**

**ApplicationContextAware在前置处理过程中**



**BeanFactoryAware**

![image-20240620173604518](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620173604518.png)

![image-20240620173855174](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620173855174.png)

![image-20240620173828296](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620173828296.png)

**setBeanFactory就是在我们具体的类中实现的，实现了BeanFactoryAware接口**。

实现 BeanFactoryAware 接口的 bean 可以在初始化时通过 setBeanFactory() 方法获取到当前 bean 所在的 BeanFactory 实例。这样就可以在 bean 内部访问和利用 BeanFactory 提供的各种功能,比如查找其他 bean、动态注册新的 bean 等。

尽管我们在学习项目的测试代码中会定义BeanFactory，并且注册并defination

但是在实际情况中，面向用户时，是使用ApplicationContext实现的，隐藏了创建BeanFactory等过程。

所以通过实现 BeanFactoryAware 接口，能够让我们在实际情况中拿到实现 BeanFactory实例。







**ApplicationContextAware**

**ApplicationContextAware**就是在我们具体的类中实现的，实现**ApplicationContextAware**接口。

在前置处理过程中

![image-20240620201920971](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620201920971.png)

调用了ApplicationContext，给bean的ApplicationContext赋值了。

在我们使用bean的时候，就可以获得ApplicationContext。

然后就可以使用ApplicationContext的功能。





# 9.bean作用域

beanDefination里面的scope字段默认是“单例”

![image-20240620203625678](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620203625678.png)

在设置scope时，先设置scope字段，再设置singleton/prototype两个字段，这两个是bool类型，一个true一个false

![image-20240620203616143](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620203616143.png)

然后在Bean生命周期中，有一些地方会判断是单例还是多例

1.如果是单例，就执行销毁方法。而多例不执行销毁方法

2.单例会放到单例对象缓存中，多例不放。

![image-20240620203348351](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240620203348351.png)





# 10.FactoryBean

spring提供了这样的一个接口：FactoryBean，可以实现它的方法，getObject()

```
FactoryBean是一种特殊的bean，当向容器获取该bean时，容器不是返回其本身，而是返回其FactoryBean#getObject方法的返回值，可通过编码方式定义复杂的bean。
```

![image-20240621172917247](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240621172917247.png)

在getBean的过程中，可以实现FactoryBean接口，来对Bean进行一些操作。

具体位置如下：

先getbean--------createBean---------docreateBean(实例化，属性赋值，前置处理，初始化方法，后置处理)

在creatBean完成后，会检查是不是FactoryBean,如果是就会调用getObject方法，完成对Bean的一些修饰。

如果不是FactoryBean，直接返回object

![image-20240621174906675](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240621174906675.png)





几个记忆的重要点，bean生命周期，应用上下文，动态代理，AOP，注解。







# 11.事件和事件监听器

首先看一下refresh方法回顾一下应用上下文流程

- 创建BeanFactory，加载BeanDefinition
- 添加ApplicationContextAwareProcessor
- 执行BeanFactoryPostProcessor
- 注册BeanPostProcessor（在我们自定义的Bean之前注册）
- **初始化事件发布者**
- **注册事件监听器**
- 提前实例化单例Bean
- **发布容器刷新完成事件**

![image-20240622174240518](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622174240518.png)

![image-20240622175144192](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622175144192.png)

传进去BeanFactory，返回得到一个applicationEventMulticaster（事件发布者对象），然后把这个“事件发布者”存到一级缓存中（hahsmap，Map<String, Object>）



然后会注册事件监听器

![image-20240622181124326](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622181124326.png)

注册事件监听器，首先会调用getBeansOfType。这个函数会把所有  **类名为：AppLicationListener**  的类实例化（docreateBean）并返回，然后把这些AppLicationListener添加到applicationEventMulticaster（事件发布者对象）中统一进行管理。





**（小总结：所谓注册register，一般都是把一些系统级别的加载器：事件监听器，前后置处理器进行一个提前实例化，并且在某个map，set里面进行保存！）**





然后发布容器刷新完成事件

在看容器发布事件之前，先看看三个事件监听器

![image-20240622193239107](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193239107.png)

![image-20240622193250068](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193250068.png)

![image-20240622193300371](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193300371.png)

都实现了ApplicationListener接口，传进去一个该监听器完成的事件。

接着看

![image-20240622193436777](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193436777.png)

完成Refresh就是靠发布一个Refresh事件

![image-20240622193506842](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193506842.png)

![image-20240622193520220](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193520220.png)

会遍历所有listener，判断是不是支持这个事件

![image-20240622193605133](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622193605133.png)

首先会用基类Object类的getclass，getGXXXXX()[0]，获得这个监听器的第一个接口名

再用这个接口，拿到第一个接口里的泛型名

然后判断事件的类名和这个事件监听器接口泛型名是否相等，如果相等，就相当于这个事件监听器实现了这个事件的处理

然后事件监听器执行操作。

![image-20240622194900115](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622194900115.png)







回到单元测试文件：

![image-20240622195043263](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622195043263.png)

这里在应用上下文加载时，发布了refresh事件，自己发布了自定义的custom事件，在钩子函数中发布了close事件

所谓发布事件，也就是执行该事件监听器的onApplicationEvent()

![image-20240622195133271](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622195133271.png)





# 12.AOP部分：切点，切点表达式

切点：方法级别，指定某个方法，或者某个类的某些方法，在这些方法上可以进行增强

切点表达式：用来指定切点。这里例子指定的切点表达式就是org.springframework.test.service.HelloService.*(..) 

​						切点就是HelloService类下所有方法

![image-20240622200400858](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622200400858.png)





# 13.JDK动态代理

JDK动态代理或者CGLIB动态代理，在创建相应的代理对象之前，都有着类似的操作。

步骤如下：

创建实现类

定义代理工厂



定义切点表达式

创建advisor

把切点表达式传给advisor

定义方法拦截器

把拦截器传给advisor



定义targetSource，把目标类传给targetSource

最后把targetSource和advisor传给代理工厂

![image-20240622210305627](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622210305627.png)

然后就创建代理类，把代理工厂传给jdk代理，然后获得代理实例

使用实例调用方法

**这里，创建好代理实例后，是通过Invoke调用目标类的方法的。**

**尽管这里在用户看来，是使用了explode方法，但是实际上是拦截器拦截到以后，回调了jdk代理的invoke方法。由这个invoke方法将我们设置的拦截器链执行，然后通过反射把原方法进行执行。**

![image-20240622210632442](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622210632442.png)

如下

![image-20240622210801990](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240622210801990.png)

然后，获得目标对象，获得拦截器链

然后前置拦截器就会先执行，再回调

后置拦截器是先回调，再执行

目标对象方法是，使用java自带的invoke进行执行。

最终的前置拦截器和后置拦截器以及目标方法全部执行。









**在实际应用中，我们主要使用的是Proxy.newProxyInstance创建JDK动态代理**

```java
UserService userService = (UserService)Proxy.newProxyInstance(service.getClass().getClassLoader(),
            service.getClass().getInterfaces(),
            new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    System.out.println("addUser()方法执行前打印日志");
                    // 调用原方法
                    Object result = method.invoke(service, args);
                    System.out.println("addUser()方法执行后打印日志");
                    return result;
                }
            });
```

用**这个方法创建动态代理类。传进去的参数是（重要）：类加载器，接口，（重写！！！！！！！！！！！）invoke方法**





# 14.CGLIB动态代理

**重点！！！！JDK实现是基于接口的，传进去的是接口类型。 CGlib传进去方法或者接口都可以运行。与基于JDK的动态代理在运行期间为接口生成对象的代理对象不同，基于CGLIB的动态代理能在运行期间动态构建字节码的class文件，为类生成子类，因此被代理类不需要继承自任何接口。**（jdk为接口生成代理对象，CGlib为类生成子类）

才明白，幡然醒悟

我们写的JDK动态代理，是实现了一个InvocationHandler接口，在生成的代理对象中的目标方法中，会调用InvocationHandler的invoke方法。







这里课程讲的比较复杂，我们先看现实中的使用：

```java
  public static UserService getGCLIBProxy(UserService service) {
        // 创建增强器
        Enhancer enhancer = new Enhancer();
        // 设置需要增强的类的对象
        enhancer.setSuperclass(service.getClass());
        // 设置回调方法
        enhancer.setCallback(new MethodInterceptor() {
            @Override
            public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy)
                throws Throwable {
                System.out.println("addUser()方法执行前打印日志");
                // 调用原方法
                Object result = methodProxy.invokeSuper(o, args);
                System.out.println("addUser()方法执行后打印日志");
                return result;
            }
        });
        UserService userService = (UserService)enhancer.create();
        return userService;
    }
```

创建enhancer，给enhancer添加目标类，**设置（重写！！！！！！！！！！！）回调方法**（ intercept）

**（注，不论是JDK动态代理还是CGlib动态代理，本身是有实现invoke方法和 intercept方法的）**





再看课程：

与JDK动态代理创建代理实例（ProxyInstance）不同，CGlib通过Enhancer创建实例的，设置了类，接口，回调函数

![image-20240624145913880](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624145913880.png)

**执行拦截器的步骤和JDK是一样的，有所不同的事，CGlib里面有个methodProxy，能够保存目标类，防止反射，从而加快速度（这里可以作为创新点，需要再学学）**

![image-20240624150258198](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624150258198.png)







# 15.代理工厂

代理工厂用于判断选择动态代理的方式。

设置了一个判断，一个变量，变量为true就选择CGlib，false选择JDK，默认为false.

![image-20240624151622858](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624151622858.png)



# 16.常用advice

框架实现了Before advice

Advice是在Join Point上执行的一个动作或者通知，一般通过拦截器调用。Spring有两大核心，IOC和AOP，在模块AOP里面有个advice。

在Spring-AOP中，增强(Advice)是如何实现的？

按照增强在目标类方法连接点的位置可以将增强划分为以下五类：

前置增强 (org.springframework.aop.BeforeAdvice) 表示在目标方法执行前来实施增强。

后置增强 (org.springframework.aop.AfterReturningAdvice)表示在目标方法执行后来实施增强。

环绕增强 (org.aopalliance.intercept.MethodInterceptor)表示在目标方法执行前后同时实施增强。

异常抛出增强 (org.springframework.aop.ThrowsAdvice) 表示在目标方法抛出异常后来实施增强。

引介增强 (org.springframework.aop.introductioninterceptor)表示在目标类中添加一些新的方法和属性。



# **17.拦截器链**

实际上，可能会设置多个拦截器，他们保存在adivceSupport的数组里面，我们在执行动态代理的回调方法的时候，会递归的按照这条拦截器链进行执行。（后置会先递归再执行，前置会先执行再递归，当size=-1，会invoke本方法，然后递归到了最深处，然后递归返回）

![image-20240624160827494](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624160827494.png)



# 18.pointcut advisor

这一讲主要讲了，advice主要是围绕pointcut进行包装的.可以在类中设置advice和pointcut。

![image-20240624170611536](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624170611536.png)

# 19.动态代理融入Bean(有点问题)

<img src="C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624163453376.png" alt="image-20240624163453376" style="zoom:67%;" />

​												如果使用这一步，就不会走下面的生命周期流程，而是生成代理对象。



在看这里的课程之前，我们回顾一下ApplicationContext的生命周期和工作流程。

首先，执行完这一句，就相当于执行到了**”使用“的步骤**，并且，bean被放到了缓存中，所谓的生命周期的主要部分已经走完了！

![image-20240624172822501](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624172822501.png)

这一句走完是这样的

![image-20240624173018074](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624173018074.png)



**然后，我们用getBean，会从缓存拿到他！不用再进行声明周期的那些初始化了！**

而本节课讲的，提前实例化单例bean时，如果发现实现实现了代理，走下面的流程，而是返回一个代理的实例。

步骤如下：

![image-20240624173719565](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624173719565.png)

![image-20240624180322592](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624180322592.png)

**这里，就说，只执行InstantiaionAwareBeanPostProcessor和后置处理器！**

![image-20240624173951888](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624173951888.png)

就是，遍历所有BeanPostProcessor，找到**InstantiaionAwareBeanPostProcessor**，调用它的实例化方法，获得代理对象进行返回。

![image-20240624174510292](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624174510292.png)

在这个方法里，会使用advisor，使用proxyFactory创建一个代理对象，并且返回。

**然后会调用后置处理器**

![image-20240624180230966](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240624180230966.png)

最后返回，总结一下，就是使用了**InstantiaionAwareBeanPostProcessor**和后置处理器，也就是跳过了初始化和前置处理等。

最终返回了一个代理对象。





# **20.**







# 21.包扫描

![image-20240625094823457](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625094823457.png)

xml文件里这样配置，会自动扫描这个目录的包

![image-20240625094901507](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625094901507.png)

会把这些扫描出来，放到beanDefination里面，类person加了component注解

![image-20240625141801457](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625141801457.png)

这种方法可以**在注册beanDefination期间，**

![image-20240625142553266](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625142553266.png)

把xml文件指定的包中的

带有component注解的类加载注册到beanDefination



**具体做法是，在解析xml文件时，会扫描带有component标签的字段，然后把包的路径拿到，进行解析，用doscan函数进行扫描。**

**doscan就是把带有component注解的类都生成beanDefination并且注册。**





# 22 @value  注解

在docreateBean里面，在实例化之后，通过beanDefination来填充属性之前，可以通过BeanPostProcessor来修改beanDefination属性，而这里使用的beanPostProcessor就是

![image-20240625151914331](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625151914331.png)

**实现了这个InstantiationAwareBeanPostprocessor接口（这个postprocessor可以实现代理方法，在xml文件中添加相关信息，并实现相关方法）的AutowiredAnnotationBeanPostprocessor,Autowired注解也是在这里实现的**

![image-20240625152047331](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625152047331.png)

![image-20240625155810774](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625155810774.png)

具体是通过这个函数，扫描一个bean所有的字段，扫描其是否有Value注解，如果有，就把里面的占位符去掉，并用properties文件里面的属性来对这个字段进行填充。

![image-20240625151820673](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625151820673.png)





# 23.@Autowired

**Autowired可以把bean里面的bean进行注入**，和value一起记忆。value用于注入属性值。

![image-20240625160337073](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625160337073.png)



会扫描所有bean的字段，如果有Autowired注解，就认为这里是一个需要注入的bean，会调用beanFactory来getbean，走docreateBean

![image-20240625161344720](C:\Users\xl\AppData\Roaming\Typora\typora-user-images\image-20240625161344720.png)
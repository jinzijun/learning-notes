### spring IOC 容器的设计
编程式获取IOC容器

```
ClassPathResource res = new ClassPathResource("applicationContext.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

在使用IOC容器时，需要如下几个步骤：
1. 创建IOC配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息。
2. 创建一个BeanFactory，这里使用DefaultListableBeanFatory;
3. 创建一个载入BeanDefinition的读取器。这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition，通过一个回调配置给BeanFactory。
4. 从定义好的资源位置读入配置信息，具体的解析过程由XmlBeanDefinitionReader来完成。完成整个载入和注册Bean定义之后，需要的IOC容器就建立起来了。这个时候就可以直接使用IOC容器了。

### ApplicationContext的应用场景
ApplicationContext提供了一下BeanFactory不具备的新特性
1. 支持不同的信息源
2. 访问资源
3. 支持应用事件。继承了接口ApplicationEventPublisher,从而在上下文中引入了事件机制。这些事件和Bean的生命周期的结合为Bean的管理提供了便利。

### ApplicationContext容器的设计原理

## IOC容器的初始化过程


### BeanFactory和FactoryBean的区别
BeanFactory是一个ioc容器。

FactoryBean是一个接口，用户可以定制实例化bean的逻辑。当在IOC容器中的Bean实现了FactoryBean后，通过getBean(String BeanName)获取到的Bean对象并不是FactoryBean的实现类对象，而是这个实现类中的getObject()方法返回的对象。要想获取FactoryBean的实现类，就要getBean(&BeanName)，在BeanName之前加上&。

FactoryBean使用举例。
```
public class Person {
    private String name;
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```
```
public class PersonFactoryBean implements FactoryBean<Person> {

    private String personInfo;

    public Person getObject() throws Exception {
        Person person = new Person();
        String[] values = StringUtils.split(personInfo, ",");
        Assert.notNull(values,"personInfo illegal");
        if(values.length < 2){
            throw new IllegalArgumentException();
        }
        person.setName(values[0]);
        person.setAge(Integer.valueOf(values[1]));
        return person;
    }


    public Class<?> getObjectType() {
        return Person.class;
    }

    public boolean isSingleton() {
        return false;
    }

    public void setPersonInfo(String personInfo) {
        this.personInfo = personInfo;
    }
}
```
```
<bean name="person" class="com.practice.jinzijun.spring.ioc.PersonFactoryBean">
        <property name="personInfo" value="jinzijun,25"/>
    </bean>
```

### lazy-init的实现
bean的依赖注入（调用bean的setter方法）的触发时机是调用BeanFactory的getBean()方法时。对于ApplicationContext，如果设置了lazy-init为false，那么依赖注入发生在容器初始化的过程中。如果设置lazy-init为true，那么依赖注入发生在容器初始化之后，第一次getBean时。
```
	public void preInstantiateSingletons() throws BeansException {

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		// 触发所有非lazy-init的单例bean的初始化
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
	}
```
### bean 初始化过程
```
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory implements AutowireCapableBeanFactory {
    protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		// 执行Aware方法
		invokeAwareMethods(beanName, bean);
		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
		    // 调用BeanPostProcessorsBeforeInitialization
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
		    // 执行init方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
	        // 执行BeanPostProcessorsAfterInitialization
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
	
	/**
	* aware方法
	*/
	private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
}
```
### autowiring（自动装配）的实现
对autowiring属性进行处理，从而完成对bean属性的自动依赖装配，是在AbstractAutowireCapableBeanFactory#populateBean中实现的。

```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
	if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// Add property values based on autowire by name if applicable.
		if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// Add property values based on autowire by type if applicable.
		if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}
		pvs = newPvs;
	}
}

protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
	for (String propertyName : propertyNames) {
		if (containsBean(propertyName)) {
			Object bean = getBean(propertyName);
			pvs.add(propertyName, bean);
			registerDependentBean(propertyName, beanName);
		}
	}
}
```

# spring AOP的实现

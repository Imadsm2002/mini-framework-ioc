````
# Compte Rendu : Inversion de Contrôle (IoC) et Injection des Dépendances (DI)

##  1. Introduction et Objectifs
L'inversion de contrôle (*Inversion of Control - IoC*) et l'injection des dépendances (*Dependency Injection - DI*) sont des mécanismes clés dans l'architecture logicielle :
* **Couplage Faible :** Garantir que les composants de haut niveau (couche métier) ne dépendent pas des classes concrètes de bas niveau (couche DAO), mais d'abstractions (interfaces).
* **Principe Open/Closed :** Permettre l'extension sans modifier le code source existant.
* **Délégation d'assemblage :** Confier l'instanciation et l'injection des composants à un conteneur IoC plutôt qu'à l'opérateur `new`[cite: 1, 10].

---

## 🛠️ 2. Partie 1 : Cas d'Étude Spring IoC

### 2.1. Conception par Interfaces (Couplage Faible)

#### Interface `IDao`
```java
package dao;

public interface IDao {
    double getData();
}

````

#### Implémentation `DaoImpl`

Java

```
package dao;
import org.springframework.stereotype.Component;

@Component("dao")
public class DaoImpl implements IDao {
    @Override
    public double getData() {
        System.out.println("Version base de données");
        return 23.5;
    }
}

```

#### Interface `IMetier`

Java

```
package metier;

public interface IMetier {
    double calcul();
}

```

#### Implémentation `MetierImpl`

Java

```
package metier;

import dao.IDao;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

@Component("metier")
public class MetierImpl implements IMetier {

    @Autowired
    @Qualifier("dao")
    private IDao dao;

    public MetierImpl() {}

    public MetierImpl(IDao dao) {
        this.dao = dao;
    }

    @Override
    public double calcul() {
        double data = dao.getData();
        return data * 2;
    }

    public void setDao(IDao dao) {
        this.dao = dao;
    }
}

```

### 2.2. Modalités d'Injection des Dépendances

#### a. Instanciation Statique

Java

```
package presentation;

import dao.DaoImpl;
import metier.MetierImpl;

public class Pres1 {
    public static void main(String[] args) {
        DaoImpl dao = new DaoImpl();
        MetierImpl metier = new MetierImpl(dao);
        System.out.println("Résultat (Statique) = " + metier.calcul());
    }
}

```

#### b. Instanciation Dynamique (API Réflexion Java)

Fichier `src/main/resources/config.txt` :

Plaintext

```
dao.DaoImpl
metier.MetierImpl

```

Classe `Pres2` :

Java

```
package presentation;

import dao.IDao;
import metier.IMetier;
import java.io.InputStream;
import java.lang.reflect.Method;
import java.util.Scanner;

public class Pres2 {
    public static void main(String[] args) throws Exception {
        InputStream is = Pres2.class.getClassLoader().getResourceAsStream("config.txt");
        Scanner scanner = new Scanner(is);

        String daoClassName = scanner.nextLine();
        Class<?> cDao = Class.forName(daoClassName);
        IDao dao = (IDao) cDao.getDeclaredConstructor().newInstance();

        String metierClassName = scanner.nextLine();
        Class<?> cMetier = Class.forName(metierClassName);
        IMetier metier = (IMetier) cMetier.getDeclaredConstructor().newInstance();

        Method setDaoMethod = cMetier.getMethod("setDao", IDao.class);
        setDaoMethod.invoke(metier, dao);

        System.out.println("Résultat (Dynamique) = " + metier.calcul());
        scanner.close();
    }
}

```

#### c. Spring IoC - Configuration XML

Fichier `src/main/resources/applicationContext.xml` :

XML

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="[http://www.springframework.org/schema/beans](http://www.springframework.org/schema/beans)"
       xmlns:xsi="[http://www.w3.org/2001/XMLSchema-instance](http://www.w3.org/2001/XMLSchema-instance)"
       xsi:schemaLocation="[http://www.springframework.org/schema/beans](http://www.springframework.org/schema/beans)
       [http://www.springframework.org/schema/beans/spring-beans.xsd](http://www.springframework.org/schema/beans/spring-beans.xsd)">

    <bean id="dao" class="dao.DaoImpl" />

    <bean id="metier" class="metier.MetierImpl">
        <property name="dao" ref="dao" />
    </bean>
</beans>

```

Classe `PresSpringXML` :

Java

```
package presentation;

import metier.IMetier;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class PresSpringXML {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
        IMetier metier = (IMetier) context.getBean("metier");
        System.out.println("Résultat (Spring XML) = " + metier.calcul());
    }
}

```

#### d. Spring IoC - Configuration par Annotations

Classe `PresSpringAnnotation` :

Java

```
package presentation;

import metier.IMetier;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class PresSpringAnnotation {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext("dao", "metier");
        IMetier metier = context.getBean(IMetier.class);
        System.out.println("Résultat (Spring Annotations) = " + metier.calcul());
    }
}

```

## ⚙️ 3. Partie 2 : Conception du Mini-Framework IoC

### 3.1. Architecture et Spécifications

Le mini-framework reproduit le comportement d'un conteneur IoC :

1. **Binding XML (JAXB) :** Chargement et désérialisation du XML de configuration.
2. **Support des Annotations :** Détection automatique des classes annotées (`@MyComponent`) et injection (`@MyAutowired`).
3. **Mécanismes d'injection supportés :**
    - Par constructeur
    - Par setter
    - Par champ direct (*Field Injection* via `setAccessible(true)`)

### 3.2. Annotations Personnalisées

- `MyComponent.java` :

Java

```
package framework.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MyComponent {
    String value() default "";
}

```

- `MyAutowired.java` :

Java

```
package framework.annotations;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.CONSTRUCTOR, ElementType.METHOD})
public @interface MyAutowired {
}

```

### 3.3. Modèle XML JAXB

Java

```
package framework.xml;

import jakarta.xml.bind.annotation.*;
import java.util.List;

@XmlRootElement(name = "beans")
@XmlAccessorType(XmlAccessType.FIELD)
public class Beans {
    @XmlElement(name = "bean")
    private List<Bean> beans;

    public List<Bean> getBeans() { return beans; }
    public void setBeans(List<Bean> beans) { this.beans = beans; }
}

```

Java

```
package framework.xml;

import jakarta.xml.bind.annotation.*;
import java.util.List;

@XmlAccessorType(XmlAccessType.FIELD)
public class Bean {
    @XmlAttribute private String id;
    @XmlAttribute(name = "class") private String className;
    @XmlElement(name = "property") private List<Property> properties;

    public String getId() { return id; }
    public String getClassName() { return className; }
    public List<Property> getProperties() { return properties; }
}

```

Java

```
package framework.xml;

import jakarta.xml.bind.annotation.*;

@XmlAccessorType(XmlAccessType.FIELD)
public class Property {
    @XmlAttribute private String name;
    @XmlAttribute private String ref;

    public String getName() { return name; }
    public String getRef() { return ref; }
}

```

### 3.4. Conteneur d'Injection : `MiniApplicationContext`

Java

```
package framework.context;

import framework.annotations.MyAutowired;
import framework.annotations.MyComponent;
import framework.xml.Bean;
import framework.xml.Beans;
import framework.xml.Property;
import jakarta.xml.bind.JAXBContext;
import jakarta.xml.bind.Unmarshaller;

import java.io.InputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.*;

public class MiniApplicationContext {
    private final Map<String, Object> context = new HashMap<>();

    // 1. Initialisation via XML (JAXB)
    public MiniApplicationContext(String xmlFileName) {
        try {
            JAXBContext jaxbContext = JAXBContext.newInstance(Beans.class);
            Unmarshaller unmarshaller = jaxbContext.createUnmarshaller();
            InputStream is = getClass().getClassLoader().getResourceAsStream(xmlFileName);
            Beans beans = (Beans) unmarshaller.unmarshal(is);

            // Phase 1 : Instanciation
            for (Bean b : beans.getBeans()) {
                Class<?> clazz = Class.forName(b.getClassName());
                Object instance = clazz.getDeclaredConstructor().newInstance();
                context.put(b.getId(), instance);
            }

            // Phase 2 : Injection des dépendances (Setter / Field)
            for (Bean b : beans.getBeans()) {
                if (b.getProperties() != null) {
                    Object target = context.get(b.getId());
                    Class<?> clazz = target.getClass();

                    for (Property prop : b.getProperties()) {
                        Object dependency = context.get(prop.getRef());
                        try {
                            String setterName = "set" + Character.toUpperCase(prop.getName().charAt(0)) + prop.getName().substring(1);
                            Method setter = clazz.getMethod(setterName, dependency.getClass().getInterfaces()[0]);
                            setter.invoke(target, dependency);
                        } catch (NoSuchMethodException e) {
                            Field field = clazz.getDeclaredField(prop.getName());
                            field.setAccessible(true);
                            field.set(target, dependency);
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 2. Initialisation via Annotations
    public MiniApplicationContext(Class<?>... packageClasses) {
        try {
            List<Class<?>> componentClasses = new ArrayList<>();
            for (Class<?> c : packageClasses) {
                if (c.isAnnotationPresent(MyComponent.class)) {
                    componentClasses.add(c);
                }
            }

            // Phase 1 : Instanciation (avec gestion du constructeur)
            for (Class<?> clazz : componentClasses) {
                MyComponent comp = clazz.getAnnotation(MyComponent.class);
                String beanId = comp.value().isEmpty() ? clazz.getSimpleName().toLowerCase() : comp.value();

                Constructor<?> autowiredCtor = null;
                for (Constructor<?> ctor : clazz.getDeclaredConstructors()) {
                    if (ctor.isAnnotationPresent(MyAutowired.class)) {
                        autowiredCtor = ctor;
                        break;
                    }
                }

                Object instance;
                if (autowiredCtor != null) {
                    Class<?> paramType = autowiredCtor.getParameterTypes()[0];
                    Object dep = findBeanByType(paramType);
                    instance = autowiredCtor.newInstance(dep);
                } else {
                    instance = clazz.getDeclaredConstructor().newInstance();
                }
                context.put(beanId, instance);
            }

            // Phase 2 : Injection par Field et Setter
            for (Object bean : context.values()) {
                Class<?> clazz = bean.getClass();

                // Injection par Field
                for (Field field : clazz.getDeclaredFields()) {
                    if (field.isAnnotationPresent(MyAutowired.class)) {
                        Object dep = findBeanByType(field.getType());
                        field.setAccessible(true);
                        field.set(bean, dep);
                    }
                }

                // Injection par Setter
                for (Method method : clazz.getDeclaredMethods()) {
                    if (method.isAnnotationPresent(MyAutowired.class) && method.getName().startsWith("set")) {
                        Object dep = findBeanByType(method.getParameterTypes()[0]);
                        method.invoke(bean, dep);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private Object findBeanByType(Class<?> type) {
        for (Object bean : context.values()) {
            if (type.isAssignableFrom(bean.getClass())) {
                return bean;
            }
        }
        return null;
    }

    public Object getBean(String id) {
        return context.get(id);
    }

    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> clazz) {
        return (T) findBeanByType(clazz);
    }
}

```

##  4. Synthèse et Conclusion

- **Rôle des interfaces :** L'encapsulation sous des interfaces permet d'interchanger les implémentations sans impact sur les couches applicatives[cite: 1, 10].
- **Intérêt de l'IoC :** La séparation entre la logique métier et l'assemblage supprime le couplage fort et facilite les tests unitaires[cite: 1, 10].
- **API Réflexion :** Le mini-framework illustre l'usage de la métaprogrammation et de l'introspection Java pour instancier et câbler les dépendances dynamiquement au runtime[cite: 1, 10].
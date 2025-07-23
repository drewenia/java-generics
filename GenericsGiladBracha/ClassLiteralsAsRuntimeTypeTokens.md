# Class Literals as Runtime-Type Tokens

JDK 5.0'daki değişikliklerden biri, `java.lang.Class` sınıfının generic olmasıdır. Bu, generic yapının bir container
sınıfı dışında kullanıldığı ilginç bir örnektir. Artık Class'ın bir type parametresi `T` olduğuna göre, `T`'nin ne
anlama geldiğini sorabilirsiniz. `T`, Class object'inin represent ettiği type'ı ifade eder.

Örneğin, `String.class` türü `Class<String>` ve `Serializable.class` türü `Class<Serializable>`'dır. Bu, reflection
kodunuzun type safety'sini artırmak için kullanılabilir.

Özellikle, `Class` içindeki `newInstance()` metodu artık `T` döndürdüğünden, object'leri reflection ile oluştururken
daha kesin type'lar elde edebilirsiniz.

Örneğin, SQL olarak verilen bir query'i gerçekleştiren ve query'e uyan database'de ki object'lerin bir collection'nını
döndüren bir helper metod yazmanız gerektiğini varsayalım.

Bir yol, factory object'ini explicitly geçirmek ve şu şekilde kod yazmaktır:

```
interface Factory<T> {
    T make();
}

public <T> Collection<T> select(Factory<T> factory, String statement){
    Collection<T> result = new ArrayList<>();
    /* jdbc kullanarak sql query calistir */
    for (/* jdbc result'ları arasinda iterate et */) {
        T item = factory.make();
        // Reflection kullanarak item'ın tüm field'larını SQL result'larından set edin.
        result.add(item);
    }
    return result;
}
```

Bunu şu şekilde çağırabilirsiniz:

```
select(new Factory<EmpInfo>(){ 
    public EmpInfo make() {
        return new EmpInfo();
    }}, "selection string");
```

ya da Factory interface'ini destekleyecek şekilde bir EmpInfoFactory sınıfı tanımlayabilirsiniz.

```
class EmpInfoFactory implements Factory<EmpInfo> {
    ...
    public EmpInfo make() { 
        return new EmpInfo();
    }
}
```

ve şu şekilde çağırabilirsiniz:

```
select(getMyEmpInfoFactory(), "selection string");
```

Bu çözümün dezavantajı, her ikisini de gerektirmesidir:

* Call site'da ayrıntılı `(verbose)` anonymous factory sınıflarının kullanımını, ya da

* kullanılan her type için bir factory sınıfı tanımlamayı ve call site'da bir factory instance’ı geçmeyi, ki bu biraz
  yapaydır.

Class literal’ını bir factory object'i olarak kullanmak doğaldır ve bu daha sonra reflection ile kullanılabilir.
Günümüzde (generics olmadan) bu kod şu şekilde yazılabilir:

```
Collection emps = sqlUtility.select(EmpInfo.class, "select * from emps");
...
public static Collection select(Class c, String sqlStatement) { 
    Collection result = new ArrayList();
    /* Run sql query using jdbc. */
    for (/* Iterate over jdbc results. */ ) { 
        Object item = c.newInstance(); 
        /* Use reflection and set all of item's
         * fields from sql results. 
         */  
        result.add(item); 
    } 
    return result; 
}
```

Ancak bu, istediğimiz kesin türde bir koleksiyon vermez. Artık Class generic olduğuna göre, bunun yerine aşağıdaki
şekilde yazabiliriz:

```
Collection<EmpInfo> emps = sqlUtility.select(EmpInfo.class, "select * from emps");
...
public static <T> Collection<T> select(Class<T> c, String sqlStatement) { 
    Collection<T> result = new ArrayList<T>();
    /* Run sql query using jdbc. */
    for (/* Iterate over jdbc results. */ ) { 
        T item = c.newInstance(); 
        /* Use reflection and set all of item's
         * fields from sql results. 
         */  
        result.add(item);
    } 
    return result; 
} 
```

Yukarıdaki kod, bize type safe bir şekilde tam olarak istediğimiz türde bir koleksiyon verir. Class literal’larını run
time type token olarak kullanma tekniği, bilinmesi gereken oldukça yararlı bir yöntemdir. Örneğin, annotation’ları
işlemek için geliştirilen yeni API’lerde bu idiom yaygın olarak kullanılmaktadır.
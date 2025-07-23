# Converting Legacy Code to Use Generics

Daha önce, new ve legacy kodun birlikte nasıl çalışabileceğini gösterdik. Şimdi, eski kodu "generic hale getirme" gibi
daha zor probleme bakalım. Legacy kodu generics kullanacak şekilde dönüştürmeye karar verirseniz, API'yi nasıl
değiştireceğinizi dikkatlice düşünmeniz gerekir.

Generic API'nin gereksiz yere kısıtlayıcı olmadığından emin olmalısınız; API'nin orijinal contract'ını desteklemeye
devam etmelidir.

`java.util.Collection`'dan bazı örneklere tekrar bakalım. Pre-generic API şu şekildedir:

```java
interface Collection {
    boolean containsAll(Collection c);

    boolean addAll(Collection c);
}
```

Onu generic hale getirmeye yönelik basit bir deneme şöyle olur:

```java
interface Collection<E> {
    boolean containsAll(Collection<E> c);

    boolean addAll(Collection<E> c);
}
```

Bu kesinlikle typesafe olsa da, API'nin orijinal contract'ını karşılamaz. `containsAll()` metodu, gelen herhangi bir
collection ile çalışır. Başarılı olur yalnızca gelen collection gerçekten sadece `E` instance'ları içeriyorsa, ancak:

* Gelen collection'ın static type'ı farklı olabilir, belki caller geçen collection'ın tam type'ını bilmiyordur ya da
  belki de `Collection<S>`’dir, burada `S` `E`'nin bir subtype'ıdır.

* `containsAll()`'ı farklı bir type collection'ı ile call etmek tamamen geçerlidir. Metod çalışmalı ve false
  döndürmelidir.

`addAll()` case'inde, `E`'nin bir subtype'ı olan instance'lardan oluşan herhangi bir collection'ı ekleyebilmeliyiz.
Bu durumu doğru şekilde nasıl handle edeceğimizi `Generic Methods` bölümünde gördük. Ayrıca, revize edilen API'nin eski
client'larla binary compability'i koruduğundan emin olmanız gerekir. Bu, API'nin erasure'ının orijinal, generic olmayan
API ile aynı olması gerektiğini ima eder. Çoğu durumda bu doğal olarak gerçekleşir, ancak bazı ince durumlar vardır.
Karşılaştığımız en ince case'lerden biri olan `Collections.max()` metodunu inceleyeceğiz.

```java
public static <T extends Comparable<? super T>> T max(Collection<T> coll) {
}
```

Bu sorun değil, ancak bu imzanın erasure'ı şudur:

```java
public static Comparable max(Collection coll) {
}
```

Bu, `max()` metodunun orijinal imzasından farklıdır:

```java
public static Object max(Collection coll) {
}
```

Formal type parametresi `T` için bound'da explicitly bir superclass belirterek erasure'ın farklı olmasını
zorlayabiliriz.

```java
public static <T extends Object & Comparable<? super T>> T max(Collection<T> coll) {
}
```

Bu, `T1 & T2 ... & Tn` syntax'ını kullanarak bir type parametresine birden fazla bound verme örneğidir. Birden fazla
bound'a sahip type variable, bound'ta listelenen tüm type'ların subtype'ı olarak bilinir. Birden fazla bound
kullanıldığında, bound'ta belirtilen ilk type, type variable'ın erasure'ı olarak kullanılır.

Son olarak, `max` sadece input collection'ınından read yaptığı için, `T`'nin herhangi bir subtype'ından oluşan
collection'lara uygulanabilir olduğunu hatırlamalıyız.

Bu bizi JDK'da kullanılan gerçek imzaya getirir:

```java
public static <T extends Object & Comparable<? super T>> T max(Collection<? extends T> coll) {
}
```

Böylesine karmaşık bir durumun pratikte nadiren ortaya çıktığı doğrudur, ancak uzman kütüphane tasarımcıları mevcut
API'leri dönüştürürken çok dikkatli düşünmeye hazır olmalıdır.

Dikkat edilmesi gereken bir diğer konu da covariant returns, yani bir subclass'da metodun return type'ını daraltmaktır.
Bu özelliği eski bir API'de kullanmamalısınız. Nedenini görmek için bir örneğe bakalım.

Orijinal API'nizin şu biçimde olduğunu varsayın:

```java
public class Foo {
    // Factory. Declare edildiği sınıfın herhangi bir instance'ını oluşturmalıdır.
    public Foo create() {
        //...
    }
}

public class Bar extends Foo {
    // Aslında bir Bar oluşturur.
    public Foo create() {
        //...
    }
}
```

Covariant returns'tan yararlanarak, bunu şu şekilde değiştirirsiniz:

```java
public class Foo {
    // Factory. Declare edildiği sınıfın herhangi bir instance'ını oluşturmalıdır.
    public Foo create() {
        //...
    }
}

public class Bar extends Foo {
    // Aslında bir Bar oluşturur.
    public Foo create() {
        //...
    }
}
```

Şimdi, kodunuzun 3rd party bir client'ının aşağıdakini yazdığını varsayın:

```
public class Baz extends Bar {
    // Aslında bir Baz oluşturur.
    public Foo create() {
        ...
    }
}
```

Java virtual machine, farklı return type'larına sahip metodların override edilmesini directly desteklemez. Bu özellik
compiler tarafından desteklenir. Sonuç olarak, `Baz` sınıfı yeniden compile edilmezse, `Bar`'ın `create()` metodunu
düzgün şekilde override edemez. Ayrıca, `Baz` değiştirilmek zorunda kalacak, çünkü yazıldığı şekilde kod
reddedilecek — `Baz`'daki `create()` metodunun return type'ı, `Bar`'daki `create()` metodunun return type'ının subtype'ı
değil.
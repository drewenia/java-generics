# Generic Methods

Bir array object ve bir collection alan ve array'de ki tüm object'leri collection'a koyan bir method yazmayı düşünün.
İşte ilk deneme:

```java
static <T> void fromArrayToCollection(T[] array, Collection<T> coll) {
    for (T t : array) {
        coll.add(t); // Correct
    }
}
```

Bu method'u, element type array'inin element type'ının supertype'ı olan herhangi bir collection ile çağırabiliriz.

```
Object[] objectArray = new Object[100];
Collection<Object> collectionObject = new ArrayList<>();

// T'nin Object olarak infer edilmesi
fromArrayToCollection(objectArray,collectionObject);

String[] stringArray = new String[100];
Collection<String> collectionString = new ArrayList<>();

// T'nin String olarak infer edilmesi
fromArrayToCollection(stringArray,collectionString);

// T'nin Object olarak infer edilmesi
fromArrayToCollection(stringArray,collectionObject);

Integer[] integerArray = new Integer[100];
Float[] floatArray = new Float[100];
Number[] numberArray = new Number[100];
Collection<Number> collectionNumber = new ArrayList<>();

// T'nin Number olarak infer edilmesi
fromArrayToCollection(integerArray,collectionNumber);

// T'nin Number olarak infer edilmesi
fromArrayToCollection(floatArray,collectionNumber);

// T'nin Number olarak infer edilmesi
fromArrayToCollection(numberArray,collectionNumber);

// T'nin Object olarak infer edilmesi
fromArrayToCollection(numberArray,collectionObject);

// COMPILE-TIME ERROR
fromArrayToCollection(numberArray,collectionString);
```

Bir generic method'a actual bir type argument geçirmemiz gerekmediğine dikkat edin. Compiler, actual argumentlerin
type'larına dayanarak bizim için type argument'i infer eder. Genellikle, call'u type açısından doğru yapacak en spesifik
type argument'i infer eder.

Ortaya çıkan bir soru şudur: generic methodları ne zaman kullanmalıyım, wildcard type'larını ne zaman kullanmalıyım?
Cevabı anlamak için, Collection kütüphanelerinden birkaç methoda bakalım.

```java
interface Collection<E> {
    boolean containsAll(Collection<?> c);

    boolean addAll(Collection<? extends E> c);
}
```

Bunun yerine generic methodlar da kullanabilirdik:

```java
interface Collection<E> {
    <T> boolean containsAll(Collection<T> c);

    <T extends E> boolean addAll(Collection<T> c);
    // Hey, type variable'lar da bound'lara sahip olabilir!
}
```

Ancak, hem `containsAll` hem de `addAll` içinde, type parameter `T` yalnızca bir kez kullanılır. Return type, type
parameter'a bağlı değildir, method'un diğer herhangi bir argument'i de (bu durumda yalnızca bir argument vardır) depend
değildir. Bu, type argument'in polymorphism için kullanıldığını bize gösterir; tek etkisi, farklı invocation
noktalarında çeşitli actual argument type'larının kullanılmasına izin vermektir. Eğer durum buysa, wildcard'lar
kullanılmalıdır. Wildcard'lar, burada ifade etmeye çalıştığımız esnek subtyping'i desteklemek için tasarlanmıştır.

Generic methodlar, bir method'un bir veya daha fazla argument'inin ve/veya return type'ının type'ları arasındaki
bağımlılıkları `(dependencies)` ifade etmek için type parameter'ların kullanılmasına olanak tanır. Böyle bir dependency
yoksa, generic method kullanılmamalıdır.

Generic methodlar ve wildcard'lar birlikte kullanılabilir. İşte `Collections.copy()` methodu:

```java
class Collections {
    public static <T> void copy(List<T> dest, List<? extends T> src) {
        //...
    }
}
```

İki parameter'ın type'ları arasındaki bağımlılığa `(dependency)` dikkat edin. Source liste olan `src`'den kopyalanan
herhangi bir object, destination liste olan `dst`'nin element türü `T`'ye assign edilebilir olmalıdır. Yani, `src`'nin
element type'ı `T`'nin herhangi bir subtype'ı olabilir — hangisi olduğu umurumuzda değil. Copy'nin imzası, bağımlılığı
`(dependency)` bir type parameter kullanarak ifade eder, ancak ikinci parameter'ın element type'ı için bir wildcard
kullanır. Bu methodun imzasını, wildcard kullanmadan tamamen başka bir şekilde de yazabilirdik:

```java
class Collections {
    public static <T, S extends T> void copy(List<T> dest, List<S> src) {
        //...
    }
}
```

Bu sorun değildir, ancak ilk type parameter hem `dst`'nin type'ında hem de ikinci type parameter olan `S`'nin bound'unda
kullanılırken, `S`'nin kendisi yalnızca bir kez, `src`'nin type'ında kullanılır — başka hiçbir şey ona bağlı `(depend)`
değildir. Bu, `S`'nin bir wildcard ile değiştirilebileceğinin bir göstergesidir. Wildcard kullanmak, explicit type
parameter'lar tanımlamaktan daha açık ve daha kısadır ve bu nedenle mümkün olan her durumda tercih edilmelidir.

Wildcard'ların ayrıca method imzalarının dışında, field'ların, local variable'ların ve array'lerin type'ları olarak da
kullanılabilmesi gibi bir avantajı vardır. İşte bir örnek.

Shape draw problemimize geri dönersek, draw isteklerinin bir history'sini tutmak istediğimizi varsayalım. History'i
Shape sınıfı içinde static bir variable olarak tutabiliriz ve `drawAll()` method'u gelen argümanı history field'ına
kaydedebilir.

```java
class Canvas {
    private final List<List<? extends Shape>> history = new ArrayList<>();

    public void draw(Shape shape) {
        shape.draw(this);
    }

    public void drawAll(List<? extends Shape> shapes) {
        history.addLast(shapes);
        for (Shape s : shapes) {
            s.draw(this);
        }
    }
}
```

Son olarak, tekrar type parameter'lar için kullanılan isimlendirme konvansiyonuna dikkat edelim. Type hakkında ayırt
edici daha spesifik bir şey olmadığında, type için `T` kullanırız. Bu, genellikle generic methodlarda böyledir.
Birden fazla type parameter varsa, alfabetikte `T`'nin yanındaki harfler, örneğin `S` kullanılabilir. Generic bir
method, generic bir sınıf içinde yer alıyorsa, karışıklığı önlemek için method ve sınıfın type parameter'ları için aynı
isimlerin kullanılmasından kaçınmak iyi bir fikirdir. Aynı durum nested generic sınıflar için de geçerlidir.
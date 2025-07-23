# More Fun with Wildcards

Bu bölümde, wildcard'ların daha ileri düzey kullanımlarını ele alacağız. Data Structure'dan okurken bounded
wildcard'ların faydalı olduğu birkaç örnek gördük. Şimdi tersini düşünelim, write-only işlemi yapılan bir data
structure. Sink interface'i bunun basit bir örneğidir.

```
interface Sink<T> {
    flush(T t);
}
```

Aşağıdaki kodda gösterildiği gibi kullanıldığını düşünebiliriz. `writeAll()` metodu, coll collection'ınında ki tüm
element'leri `snk`'ye write etmek ve write edilen son element'i döndürmek için tasarlanmıştır.

```
public static <T> T writeAll(Collection<T> coll, Sink<T> snk) {
    T last;
    for (T t : coll) {
        last = t;
        snk.flush(last);
    }
    return last;
}

...
Sink<Object> s;
Collection<String> cs;
String str = writeAll(cs, s); // Illegal call.
```

Yazıldığı gibi, `writeAll()` call'u illegal'dir çünkü geçerli bir type argümanı infer edilemez; Ne String ne de Object,
`T` için uygun type'lardır çünkü Collection element'i ile Sink element'i aynı type'da olmalıdır. Bu hatayı, `writeAll()`
imzasını aşağıda gösterildiği gibi wildcard kullanarak düzeltebiliriz.

```
public static <T> T writeAll(Collection<? extends T>, Sink<T>) {...}
...
// Call is OK, but wrong return type. 
String str = writeAll(cs, s);
```

Call artık legal, ancak assignment hatalıdır çünkü döndürülen type `Object` olarak infer edilir; çünkü `T`, `s`'nin
element type'ı olan `Object` ile eşleşir.

Çözüm, henüz görmediğimiz bir bounded wildcard formunu kullanmaktır: lower bound wildcard'lar. `? super T` syntax'ı,
`T`'nin supertype'ı olan (ya da `T`'nin kendisi; supertype ilişkisi reflexive'dir) unknown bir type'ı ifade eder.
Bu, daha önce kullandığımız bounded wildcard'ların tersidir; `? extends T`, `T`'nin subtype'ı olan unknown bir type'ı
ifade etmek için kullanılır.

```
public static <T> T writeAll(Collection<T> coll, Sink<? super T> snk) {
    ...
}
String str = writeAll(cs, s); // Yes! 
```

Bu syntax'ı kullanarak, call legal olur ve infer edilen type istediğimiz gibi String olur.

Şimdi daha gerçekçi bir örneğe geçelim. `java.util.TreeSet<E>`, ordered `E` type elementlerden oluşan bir tree represent
eder. TreeSet oluşturmanın bir yolu, constructor'a `Comparator` object'i geçirmektir. Bu comparator, TreeSet
elementlerini istenilen order'a göre sort etmek için kullanılır.

```
TreeSet(Comparator<E> c)
```

Comparator interface'i temel olarak şudur:

```
interface Comparator<T> {
    int compare(T first, T second);
}
```

Diyelim ki `TreeSet<String>` oluşturmak ve uygun bir comparator geçirmek istiyoruz. Ona Stringleri compare edebilen bir
Comparator vermemiz gerekir. Bu `Comparator<String>` ile yapılabilir, ancak `Comparator<Object>` da aynı işi görür.
Ancak, yukarıda verilen constructor'ı `Comparator<Object>` üzerinde invoke edemeyiz. İstediğimiz esnekliği elde etmek
için `lower bounded wildcard` kullanabiliriz:

```
TreeSet(Comparator<? super E> c)
```

Bu kod, herhangi bir applicable comparator'ın kullanılmasına izin verir.

Lower bounded wildcard'ların kullanımına dair son bir örnek olarak, kendisine argüman olarak verilen collection'da ki
maximal elementi döndüren `Collections.max()` metoduna bakalım. Şimdi, `max()` metodunun çalışabilmesi için, kendisine
verilen collection'da ki tüm element'ler `Comparable`'ı implement etmeli. Ayrıca, hepsi birbirleriyle comparable olmalı.

Bu metod imzasını generic hale getirmeye yönelik ilk deneme şudur:

```
public static <T extends Comparable<T>> T max(Collection<T> coll)
```

Yani, metod kendisiyle comparable olan bazı `T` type'ında bir collection alır ve o type'dan bir element döner. Ancak, bu
kod aşırı kısıtlayıcı olur. Nedenini görmek için, rastgele object'lerle comparable bir type'ı düşünelim:

```
class Foo implements Comparable<Object> {
    ...
}
Collection<Foo> cf = ... ;
Collections.max(cf); // Should work.
```

`cf` içindeki her element, `cf` içindeki diğer her element ile compare edilebilir, çünkü her böyle element bir Foo'dur
ve Foo herhangi bir object ile, özellikle başka bir Foo ile compare edilebilir.

Ancak, yukarıdaki imza kullanıldığında, call reject edilir. Infer edilen type `Foo` olmalı, fakat Foo `Comparable<Foo>`
implement etmez. `T`'nin tam olarak kendisiyle comparable olması gerekli değildir. Gerekli olan, `T`'nin
supertype'larından biriyle comparable olmasıdır. Bu bize şunu verir:

```
public static <T extends Comparable<? super T>> T max(Collection<T> coll)
```

`Collections.max()` metodunun gerçek imzasının daha karmaşık olduğunu unutmayın. Bu mantık, rastgele type'lar için
çalışması amaçlanan Comparable kullanımının neredeyse tamamı için geçerlidir: Her zaman `Comparable <? super T>`
kullanmak istersiniz.

Genel olarak, eğer bir API sadece bir type parametresi `T`'yi argüman olarak kullanıyorsa, kullanımları lower bounded
wildcard'lar `(? super T)` avantajından yararlanmalıdır. Buna karşılık, API sadece `T` döndürüyorsa, upper bounded
wildcard'lar `(? extends T)` kullanarak client'larınıza daha fazla esneklik sağlarsınız.

## Wildcard Capture

```
Set<?> unknownSet = new HashSet<String>();
...
/* Add an element  t to a Set s. */ 
public static <T> void addToSet(Set<T> s, T t) {
    ...
}
```

Aşağıdaki call illegaldir.

```
addToSet(unknownSet, "abc"); // Illegal.
```

Gerçekten geçirilen set'in string'lerden oluşması fark etmez; Önemli olan, argüman olarak geçirilen expression'ın
unknown bir type'da bir set olmasıdır; bu da set'in string ya da herhangi custom bir type olduğu garanti edilemez.

Şimdi, aşağıdaki koda bakalım:

```
class Collections {
    ...
    <T> public static Set<T> unmodifiableSet(Set<T> set) {
        ...
    }
}
...
Set<?> s = Collections.unmodifiableSet(unknownSet); // This works! Why?
```

Bu izin verilmemeli gibi görünüyor; ancak, bu spesifik çağrıya bakıldığında, izin vermek tamamen safe. Nihayetinde,
`unmodifiableSet()` element type'ından bağımsız olarak herhangi bir Set için çalışır.

Bu durum nispeten sık ortaya çıktığından, kodun safe olduğu kanıtlanabilen çok özel durumlarda böyle koda izin veren
özel bir kural vardır. Wildcard capture olarak bilinen bu kural, compiler'ın wildcard'ın bilinmeyen type'ını generic
metoda type argümanı olarak infer etmesine izin verir.
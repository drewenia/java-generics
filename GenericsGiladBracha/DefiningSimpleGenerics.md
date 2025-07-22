# Defining Simple Generics

İşte `java.util` paketindeki List ve Iterator interface’lerinin tanımlarından küçük bir alıntı:

```java
public interface List<E> {
    void add(E x);

    Iterator<E> iterator();
}

public interface Iterator<E> {
    E next();

    boolean hasNext();
}
```

Bu code’un tamamı tanıdık olmalı, açılı parantez `<>` içindekiler hariç. Bunlar, List ve Iterator interface’lerinin
formal type parameter’larının declaration’larıdır. Type parameter’lar generic declaration boyunca, ordinary type'ları
kullanacağınız hemen hemen her yerde kullanılabilir.

Introduction'da, `List<Integer>` gibi generic type declaration `List`’in invocation’larını gördük. Invocation’da
(genellikle parameterized type olarak adlandırılır), formal type parameter’ın (bu durumda `E`) tüm kullanımları actual
type argument (bu durumda `Integer`) ile değiştirilir.

`List<Integer>`’in, `E`’nin tamamen `Integer` ile replace edildiği bir List versiyonu olduğunu düşünebilirsiniz:

```java
interface IntegerList {
    void add(Integer x);

    Iterator<Integer> iterator();
}
```

Bu sezgi faydalı olabilir, ancak yanıltıcıdır. Faydalıdır, çünkü parameterized type `List<Integer>` gerçekten de bu
expansion'a çok benzeyen method’lara sahiptir. Yanıltıcıdır, çünkü bir generic’in declaration’ı aslında hiçbir zaman bu
şekilde genişletilmez.

Code’un birden fazla kopyası yoktur — ne source'da, ne binary’de, ne diskte ne de memory’de. Bir generic type
declaration bir kez compile olur ve tıpkı ordinary bir class veya interface declaration’ı gibi tek bir class dosyasına
dönüştürülür. Type parameter’lar, method veya constructor’larda kullanılan sıradan parameter’lara benzer.

Nasıl ki bir method, üzerinde operates yaptığı value’ların type'larını tanımlayan formal value parameter’lara sahipse,
bir generic declaration da formal type parameter’lara sahiptir. Bir method invoke edildiğinde, actual argument’ler
formal parameter’ların yerine geçer ve method body'si değerlendirilir `(evaluated)`.

Bir generic declaration invoke edildiğinde, actual type argument’ler formal type parameter’ların yerine geçer.
Adlandırma kuralları hakkında bir not: Formal type parameter’lar için özlü (mümkünse tek karakterli) ancak anlamlı
isimler kullanmanızı öneririz. Bu isimlerde küçük harf karakterlerden kaçınmak en iyisidir; bu, formal type
parameter’ları sıradan class ve interface’lerden ayırt etmeyi kolaylaştırır. Pek çok container type, element için E
kullanır, yukarıdaki örneklerde olduğu gibi. Daha sonra bazı ek kuralları göreceğiz.
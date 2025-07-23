# Interoperating with Legacy Code

Şimdiye kadar, tüm örneklerimiz herkesin generic destekleyen Java programlama dilinin en son sürümünü kullandığı
idealize bir dünyayı varsayıyordu. Ne yazık ki, gerçekte durum böyle değil. Milyonlarca satır kod dilin önceki
sürümlerinde yazıldı ve hepsi bir gecede dönüştürülmeyecek. Bu bölümde, daha basit bir probleme odaklanacağız: legacy
kod ile generic kod nasıl birlikte çalışabilir? Bu sorunun iki kısmı vardır: generic kod içinde legacy kod kullanmak ve
legacy kod içinde generic kod kullanmak.

## Using Legacy Code in Generic Code

Kendi kodunuzda generic avantajlarından yararlanırken eski kodu nasıl kullanabilirsiniz? Bir örnek olarak,
`com.Example.widgets` package'ini kullanmak istediğinizi varsayalım. `Example.com`’daki kişiler, aşağıda öne çıkanları
gösterilen bir envanter kontrol sistemi pazarlamaktadır:

```
package com.Example.widgets;

interface Part {//...}

public class Inventory {
    /**
     * Inventory database'ine yeni bir Assembly ekler
     * Assembly’ye name adı verilir ve parts ile belirtilen bir part setinden oluşur.
     * parts collection'ınında ki tüm elementler Part interface'ini desteklemelidir.
     **/ 
    public static void addAssembly(String name, Collection parts) {//...}
    public static Assembly getAssembly(String name) {...}
}

public interface Assembly {
    // Returns a collection of Parts
    Collection getParts();
}
```

Şimdi, yukarıdaki API'yi kullanan yeni kod eklemek istiyorsunuz. `addAssembly()`'yi her zaman doğru argümanlarla
çağırdığınızdan emin olmak iyi olur - yani, geçirdiğiniz Collection'ın gerçekten Part type'ında bir Collection olması
gerekir. Elbette, generics bunun için biçilmiş kaftandır:

```
package com.mycompany.inventory;

import com.Example.widgets.*;

public class Blade implements Part {
    ...
}

public class Guillotine implements Part {
}

public class Main {
    public static void main(String[] args) {
        Collection<Part> c = new ArrayList<Part>();
        c.add(new Guillotine()) ;
        c.add(new Blade());
        Inventory.addAssembly("thingee", c);
        Collection<Part> k = Inventory.getAssembly("thingee").getParts();
    }
}
```

`addAssembly` çağrıldığında, ikinci parametrenin Collection türünde olması beklenir. Actual argüman `Collection<Part>`
türündedir. Bu çalışır, ama neden? Sonuçta, çoğu Collection `Part` object'leri içermez ve bu yüzden genel olarak
compiler, Collection türünün ne tür bir koleksiyona işaret ettiğini bilemez. Doğru generic kodda, Collection her zaman
bir type parametresiyle birlikte olur. Collection gibi generic bir type parametre olmadan kullanıldığında, buna raw type
denir.

Çoğu kişinin ilk varsayımı Collection'ın aslında `Collection<Object>` anlamına geldiğidir. Ancak, daha önce gördüğümüz
gibi, `Collection<Object>` gereken bir yere `Collection<Part>` geçirmek güvenli değildir. Collection türünün,
`Collection<?>` gibi, unknown bir türde collection'ı ifade ettiğini söylemek daha doğrudur.

Ama bekleyin, bu da doğru olamaz! Collection döndüren `getParts()` call'unu düşünün. Bu, `Collection<Part>` olan `k`'ya
assign edilir. Call'un sonucu `Collection<?>` ise, assignment hata olurdu.

Gerçekte, assignment legaldir ancak unchecked uyarısı oluşturur. Uyarı gereklidir çünkü compiler doğruluğunu garanti
edemez. `getAssembly()` içindeki legacy kodu, döndürülen collection'ın gerçekten `Part` collection'ı olduğunu kontrol
etmenin bir yolu yoktur. Kodda kullanılan type Collection'dır ve böyle bir collection'a her türlü object legal olarak
eklenebilir.

Peki, bu bir hata olmamalı mı? Teorik olarak evet; ancak pratikte, generic kod legacy kodu çağıracaksa, buna izin
verilmesi gerekir. Bu durumda, type signature bunu göstermese bile `getAssembly()` kontratının bir `Part` collection'ı
döndürdüğünü belirttiği için, assignment'ın güvenli olduğuna kendinizin, yani programcının, emin olması gerekir.

Yani raw type'lar wildcard type'lara oldukça benzer, ancak onlar kadar sıkı şekilde typecheck edilmezler. Bu,
generics'in önceden var olan legacy kodla birlikte çalışmasına izin vermek için bilinçli bir tasarım kararıdır.

Generic koddan legacy kod call etmek doğası gereği tehlikelidir; generic kod ile non-generic legacy kodu karıştırdığınız
anda, generic type sisteminin normalde sağladığı tüm güvenlik garantileri geçersiz olur. Yine de, generics hiç
kullanmamış olmaktan daha iyi bir durumdasınız. En azından kendi tarafınızdaki kodun tutarlı olduğunu biliyorsunuz.

Şu anda, dışarıda generic koda kıyasla çok daha fazla non-generic kod var ve kaçınılmaz olarak bu ikisinin mix edilmesi
gereken durumlar olacaktır.

Legacy ve generic kodu mix etmeniz gerektiğini fark ederseniz, unchecked uyarılarına dikkat edin. Uyarıya neden olan
kodun güvenliğini nasıl gerekçelendirebileceğinizi dikkatlice düşünün.

Peki ya hâlâ bir hata yaptıysanız ve uyarıya neden olan kod gerçekten type safe değilse ne olur? Böyle bir duruma
bakalım. Bu süreçte, compiler'ın işleyişi hakkında da biraz fikir edineceğiz.

## Erasure and Translation

```java
public String loophole(Integer x) {
    List<String> ys = new LinkedList<String>();
    List xs = ys;
    xs.add(x); // Compile-time unchecked warning
    return ys.iterator().next();
}
```

Burada, bir string listesi ile klasik bir listeyi aynı ad altında göstermiş olduk. Listeye bir Integer ekliyoruz ve
ardından bir String extract etmeye çalışıyoruz. Bu açıkça yanlıştır. Eğer uyarıyı görmezden gelip bu kodu çalıştırırsak,
yanlış türü kullanmaya çalıştığımız noktada tam olarak hata verir. Bu kod, runtime da şu şekilde davranır:

```
public String loophole(Integer x) {
    List ys = new LinkedList;
    List xs = ys;
    xs.add(x); 
    return(String) ys.iterator().next(); // run time error
}
```

Listeden bir element extract ettiğimizde ve onu String olarak cast ederek String gibi işlemeye çalıştığımızda,
`ClassCastException` alırız. `loophole()`'un generic versiyonunda da tam olarak aynı şey olur.

Bunun nedeni, generics'in Java compiler tarafından erasure adı verilen bir front-end conversion olarak implement
edilmesidir. Bunu (neredeyse) generic `loophole()`'un non-generic versiyona source-to-source translation olarak
düşünebilirsiniz.

Sonuç olarak, unchecked uyarılar olsa bile Java virtual machine'in type safety ve bütünlüğü asla risk altında olmaz.

Temelde, erasure tüm generic type bilgisini kaldırır (veya siler). Köşeli parantezler `<>` arasındaki tüm type bilgisi
atılır; örneğin, `List<String>` gibi parameterized bir type List'e convert edilir.

Kalan tüm type variable kullanımları, type variable’ın upper bound'u ile (genellikle Object) değiştirilir. Ve ortaya
çıkan kod type açısından doğru olmadığında, `loophole`'un son satırında olduğu gibi uygun type'a cast eklenir.
Erasure'ın tam detayları bu tutorial kapsamı dışındadır, ancak az önce verdiğimiz basit açıklama gerçeğe çok yakındır.

## Using Generic Code in Legacy Code

Şimdi ters durumu düşünelim. Example.com API'lerini generic kullanacak şekilde dönüştürmeye karar verdi, ancak bazı
client'lar henüz dönüştürmedi. Böylece kod şimdi şöyle görünüyor:

```
package com.Example.widgets;

public interface Part { 
    ...
}

public class Inventory {
    /**
     * Inventory database'ine yeni bir Assembly ekler
     * Assembly’ye name adı verilir ve parts ile belirtilen bir part setinden oluşur.
     * parts collection'ınında ki tüm elementler Part interface'ini desteklemelidir.
     **/ 
    public static void addAssembly(String name, Collection<Part> parts) {...}
    public static Assembly getAssembly(String name) {...}
}

public interface Assembly {
    // Returns a collection of Parts
    Collection<Part> getParts();
}
```

ve client kodu şöyle görünüyor:

```
package com.mycompany.inventory;

import com.Example.widgets.*;

public class Blade implements Part {
...
}

public class Guillotine implements Part {
}

public class Main {
    public static void main(String[] args) {
        Collection c = new ArrayList();
        c.add(new Guillotine()) ;
        c.add(new Blade());

        // 1: unchecked warning
        Inventory.addAssembly("thingee", c);

        Collection k = Inventory.getAssembly("thingee").getParts();
    }
}
```

Client kodu generics kullanılmadan önce yazıldı, ancak `com.Example.widgets` paketini ve generic type kullanan
collection kütüphanesini kullanıyor. Client kodundaki tüm generic type deklarasyonları raw type'lardır. 1. satır
unchecked uyarısı oluşturur, çünkü raw Collection, Part collection'u beklenen yere geçirilir ve compiler raw
Collection'ın gerçekten Part collection'ı olduğunu garanti edemez.
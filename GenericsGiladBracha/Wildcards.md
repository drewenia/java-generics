# Wildcards

Bir Collection'da ki tüm element’leri yazdıran bir rutin yazma sorununu düşünün. İşte bunu dilin daha eski bir
versiyonunda (yani 5.0 öncesi bir sürümde) nasıl yazabileceğiniz:

```java
void printCollection(Collection c) {
    Iterator i = c.iterator();
    for (k = 0; k < c.size(); k++) {
        System.out.println(i.next());
    }
}
```

Ve işte bunu generics kullanarak (ve yeni for loop syntax'ı ile) yazmaya yönelik basit bir girişim:

```java
void printCollection(Collection<Object> c) {
    for (Object e : c) {
        System.out.println(e);
    }
}
```

Sorun şu ki, bu yeni versiyon eskisinden çok daha az kullanışlıdır. Eski kod herhangi bir Collection type'ı ile
parametre olarak call edilebilirken, yeni kod sadece `Collection<Object>` alır ki, az önce gösterdiğimiz gibi, bu tüm
Collection türlerinin supertype’ı `değildir`!

Peki tüm Collection type'larının supertype’ı nedir? `Collection<?>` olarak yazılır (okunuşu "collection of unknown"),
yani element type'ı herhangi bir şeyle match olan bir Collection. Açık nedenlerle buna wildcard type denir. Şöyle
yazabiliriz:

```java
void printCollection(Collection<?> coll) {
    for (Object e : coll) {
        System.out.println(e);
    }
}
```

ve şimdi, onu herhangi bir tür Collection ile call edebiliriz. `printCollection()` içinde, `coll`’den element’leri hâlâ
okuyabildiğimizi ve onlara `Object` type'ı verebildiğimizi unutmayın. Bu her zaman safe’dir, çünkü Collection'ın actual
type'ı ne olursa olsun, içinde object’ler bulunur. Ancak içine rastgele object’ler eklemek safe değildir:

```
Collection<?> c = new ArrayList<String>();
c.add(new Object()); // Compile time error
```

`c`’nin element type'ının neyi represent ettiğini bilmediğimiz için, ona object ekleyemeyiz. `add()` method’u,
Collection'ın element type'ı olan `E` type'ında argument alır. Actual type parameter `?` olduğunda, bu unknown bir
type'ı represent eder. `add`’e geçeceğimiz herhangi bir parameter, bu unknown type'ın bir subtype’ı olmak zorundadır.
Bu type'ın ne olduğunu bilmediğimiz için hiçbir şey geçemeyiz. Tek istisna, her türün bir member’ı olan `null`’dır.
Öte yandan, elimizde bir List<`?`> olduğunda, `get()` call edebilir ve result'ını kullanabiliriz. Result type unknown
bir type'dır, ancak her zaman bir `object` olduğunu biliriz.

Bu nedenle, `get()` result'ını `Object` type'ında bir variable’a assign etmek ya da Object type'ının beklendiği bir
yerde parameter olarak geçmek safe’tir.

```java
public static void main(String[] args) {
    List<String> strList = List.of("Ocean", "Joe", "Foo", "Bar");
    print(strList); // => Ocean, Joe, Foo, Bar
}

static void print(Collection<?> coll) {
    for (Object elem : coll) {
        System.out.println(elem);
    }
}
```

## Bounded Wildcards

Rectangle ve circle'lar gibi shape'ler çizebilen basit bir çizim uygulamasını düşünün. Bu shape'leri program içinde
represent etmek için şu şekilde bir class hiyerarşisi tanımlayabilirsiniz:

```java
abstract class Shape {
    abstract void draw(Canvas c);
}

class Circle extends Shape {
    private int x, y, radius;

    @Override
    void draw(Canvas c) {
        // ...
    }
}

class Rectangle extends Shape {
    private int x, y, width, height;

    @Override
    void draw(Canvas c) {
        // ...
    }
}
```

Bu class'lar bir canvas üzerinde çizilebilir:

```java
class Canvas {
    public void draw(Shape shape) {
        shape.draw(this);
    }
}
```

Herhangi bir draw genellikle birden fazla shape içerir. Bunların bir list olarak represent edildiği varsayılırsa,
hepsini draw eden bir method'un Canvas içinde olması kullanışlı olur:

```java
class Canvas {
    public void draw(Shape shape) {
        shape.draw(this);
    }

    public void drawAll(List<Shape> shapes) {
        for (Shape s : shapes) {
            s.draw(this);
        }
    }
}
```

Şimdi, type kuralları `drawAll()` methodunun yalnızca tam olarak `Shape` listesinin üzerinde call edilebileceğini
söyler: Örneğin, `List<Circle>` üzerinde call edilemez. Bu talihsiz bir durumdur, çünkü method sadece listeden
shape'leri okur, dolayısıyla `List<Circle>` üzerinde de call edilebilirdi. Gerçekten istediğimiz, method'un herhangi bir
türden shape listesi kabul etmesidir:

```java
class Canvas {
    public void draw(Shape shape) {
        shape.draw(this);
    }

    public void drawAll(List<? extends Shape> shapes) {
        // ...
    }
}
```

Burada küçük ama çok önemli bir fark var: `List<Shape>` type'ını `List<? extends Shape>` ile değiştirdik. Artık
`drawAll()` Shape'in herhangi bir subclass'ının listesini kabul edecek, bu yüzden isterseniz `List<Circle>` üzerinde
çağırabiliriz.

`List<? extends Shape> bounded wildcard` örneğidir. `?` unknown bir type'ı represent eder, daha önce gördüğümüz
wildcard'lar gibi. Ancak, bu durumda bu unknown type'ın aslında `Shape`'in bir subtype'ı olduğunu biliyoruz. (Not:
Shape'in kendisi veya bir subclass'ı olabilir; kelimenin tam anlamıyla Shape'i extend etmek zorunda değildir).
Wildcard'ın `upper bound`'unun `Shape` olduğunu söyleriz.

Wildcard kullanımının sağladığı esneklik için, her zamanki gibi bir bedel ödenir. Ödenen bedel, artık method body'sinde
shape'lere write etmenin yasak olmasıdır. Örneğin, bu izin verilmez:

```java
public void addRectangle(List<? extends Shape> shapes) {
    // Compile-time error!
    shapes.add(0, new Rectangle());
}
```

Yukarıdaki kodun neden yasak olduğunu kendiniz anlayabilirsiniz. `shapes.add()` ifadesinin ikinci parametresinin
type'ı `?` extends Shape'tir — yani Shape'in unknown bir subtype'ıdır. Hangi type olduğunu bilmediğimiz için,
Rectangle'ın bir supertype'ı olup olmadığını da bilemeyiz; böyle bir supertype olabilir de olmayabilir de, bu yüzden
oraya bir Rectangle geçirmek safe değildir.

Bounded wildcard'lar, DMV'nin verilerini nüfus dairesine iletmesi örneğini handle etmek için tam olarak gereken şeydir.
Örneğimiz, data'ların isimlerden (string olarak represent edilen) insanlara (Person gibi reference type'lar veya onun
subtype'ları olan Driver gibi) yapılan bir mapping ile represent edildiğini varsayar. `Map<K,V>`, map'in key'lerini ve
value'larını represent eden iki type argümanı alan bir generic type örneğidir.

Yine, formal type parameter'ları için adlandırma kuralına dikkat edin—key'ler için K ve value'lar için V kullanılır.

```
public class Census {
    public static void addRegistry(Map<String, ? extends Person> registry) {
}
...

Map<String, Driver> allDrivers = ... ;
Census.addRegistry(allDrivers);
```

Upper Bound Wildcard Example;

```java
public static void main(String[] args) {
    List<Integer> intList = Arrays.asList(1, 2, 3, 4, 5);
    double sum = sum(intList);
    System.out.println(sum); // => 15.0

    List<Double> doubleList = Arrays.asList(1.2, 2.3, 3.5);
    double sum1 = sum(doubleList);
    System.out.println(sum1); // => 7.0
}

public static double sum(List<? extends Number> numbers) {
    double sum = 0.0D;
    for (Number num : numbers) {
        sum += num.doubleValue();
    }
    return sum;
}
```

## Lower Bound Wildcard

Java generic'lerinde, lower bound wildcard bir generic type'ın belirli bir type veya onun herhangi bir supertype'ı
olabileceğini belirtmenizi sağlar. `? super` keyword'u ile ifade edilir. Bu, bir collection'a element eklemek
istediğinizde özellikle kullanışlıdır; çünkü collection'ın belirtilen type'tan veya onun herhangi bir superclass'ından
object'leri tutabilmesini garanti eder.

`? super T`, "`T` veya `T`'nin herhangi bir superclass'ı olan herhangi bir type" anlamına gelir.

Elinizde `List<? super Integer>` varsa, bu `List<Integer>`, `List<Number>`, `List<Object>` gibi type'lara refer
edebilir.

* Temel use case : Element Ekleme

Lower bounded wildcard kullanmanın temel nedeni, bir collection'a element eklemek istemenizdir. Elinizde bir
`List<? super Integer>` varsa, ona bir Integer (veya Integer'ın herhangi bir subtype'ı) ekleyebilirsiniz; çünkü Integer
her zaman Integer, Number veya Object ile uyumludur.

```java
class Animal{
    @Override
    public String toString(){
        return "Animal";
    }
}

class Cat extends Animal{
    @Override
    public String toString() {
        return "Cat";
    }
}

class SiameseCat extends Cat{
    @Override
    public String toString() {
        return "Siemese cat";
    }
}
```

Şimdi, element eklemek için lower bound wildcard kullanan bir method oluşturalım:

```java
class LowerBoundWildcardExample{
    public static void main(String[] args) {
        List<Animal> animals = new ArrayList<>();
        animals.add(new Cat());
        animals.add(new SiameseCat());
        addCats(animals);
    }

    public static void addCats(List<? super Cat> catList){
        for (Object object : catList){
            System.out.println(object); // => Cat, Siamese cat
        }
    }
}
```
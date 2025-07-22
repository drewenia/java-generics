# Introduction

JDK 5.0, Java programming language'a birkaç yeni extension getirir. Bunlardan biri generic'lerin tanıtılmasıdır. Bu
trail, generic'lere bir giriş niteliğindedir. Diğer dillerde, özellikle C++ template'lerinden benzer yapılarla aşina
olabilirsiniz. Eğer öyleyse, hem benzerlikler hem de önemli farklılıklar olduğunu göreceksiniz. Eğer başka yerlerden
benzer yapılara aşina değilseniz, bu daha da iyidir; herhangi bir yanlış anlamayı unutmadan sıfırdan başlayabilirsiniz.

Generic'ler, type'lar üzerinde abstraction yapmanızı sağlar. En yaygın örnekler, Collections hiyerarşisindeki container
type'larıdır.

İşte bu tür bir kullanım:

```
List myIntList = new LinkedList(); // 1
myIntList.add(0); // 2
Integer x = (Integer) myIntList.iterator().next(); // 3
```

Satır 3 de ki cast biraz rahatsız edici. Genellikle, programmer belirli bir listeye ne tür data yerleştirildiğini bilir.
Ancak, cast zorunludur. Compiler yalnızca iterator tarafından bir `Object` döndürüleceğini garanti edebilir. Integer
türündeki bir variable’a assignment’ın typesafe olmasını sağlamak için cast gereklidir.

Elbette, cast sadece karmaşıklık yaratmaz. Ayrıca programmer’ın hata yapabileceği için bir runtime hatası olasılığını da
getirir. Ya programmer’lar gerçekten niyetlerini ifade edebilse ve bir listeyi belirli bir data türünü içerecek şekilde
kısıtlayabilse? Bu, generics’in temel fikridir. İşte yukarıda verilen program parçasının generics kullanılarak
hazırlanmış bir versiyonu:

```
List<Integer> myIntList = new LinkedList<>(); // 1
myIntList.add(0); // 2
Integer x = myIntList.iterator().next(); // 3
```

`myIntList` variable’ı için yapılan type declaration’a dikkat edin. Bu, bunun sadece rastgele bir List olmadığını, aynı
zamanda `List<Integer>` olarak yazılan bir Integer List olduğunu belirtir. List’in bir generic interface olduğunu ve bir
type parameter aldığını söyleriz — bu durumda Integer. List object’i oluştururken de bir type parameter belirtiriz.
Ayrıca, 3. satırdaki cast’in de kalktığını unutmayın.

Şimdi, başardığımız şeyin sadece karmaşıklığı başka bir yere taşımak olduğunu düşünebilirsiniz. 3. satırdaki Integer
cast’i yerine, 1. satırda bir type parameter olarak Integer var. Ancak burada çok büyük bir fark var. Compiler artık
programın type doğruluğunu compile-time’da kontrol edebilir.

`myIntList`’in `List<Integer>` türüyle declare edildiğini söylediğimizde, bu bize `myIntList` variable’ı hakkında,
nerede ve ne zaman kullanılırsa kullanılsın `true` olacak bir şey söyler ve compiler bunu garanti eder. Buna karşılık,
cast bize programmer’ın kodun tek bir noktasında doğru olduğunu düşündüğü bir şeyi söyler. Özellikle büyük programlarda,
net sonuç okunabilirlik ve sağlamlıkta artıştır.
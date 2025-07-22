# Generics and Subtyping

Generics konusundaki anlayışınızı test edelim. Aşağıdaki code snippet legal mı?

```java
List<String> ls = new ArrayList<>(); // 1
List<Object> lo = ls; // 2 COMPILER-ERROR
```

Satır 1 kesinlikle legal'dir. Sorunun daha zor kısmı 2. satırdır. Bu, String List’in Object List olup olmadığı sorusuna
indirgenir. Çoğu kişi içgüdüsel olarak “Tabii ki!” diye cevap verir.

Şimdi, sonraki birkaç satıra bakın:

```
lo.add(new Object()); // 3
String s = ls.get(0); // 4 Bir Object’i String’e atamaya çalışıyor!
```

Burada `ls` ve `lo`’ya alias verdik. `ls`’ye, yani String listesine, `lo` alias’ı üzerinden erişerek içine rastgele
object’ler ekleyebiliriz. Sonuç olarak `ls` artık sadece String’ler tutmaz ve içinden bir şey almaya çalıştığımızda
tatsız bir sürprizle karşılaşırız. Java compiler bunun olmasını elbette engeller. 2. satır compile time hatası
verecektir.

Genel olarak, eğer `Foo`, `Bar`’ın subtype’ı (subclass veya subinterface) ise ve `G` bir generic type declaration ise,
`G<Foo>`’nun `G<Bar>`’nin subtype’ı olduğu doğru değildir. Bu, generics hakkında öğrenmeniz gereken en zor şeylerden
biridir, çünkü derinlemesine yerleşmiş sezgilerimize ters düşer.

Collection'ların değişmediğini varsaymamalıyız. Sezgimiz, bunları immutable olarak düşünmemize yol açabilir. Örneğin,
motorlu taşıtlar departmanı nüfus bürosuna bir sürücü listesi sağlarsa, bu mantıklı görünür. Driver’ın Person subtype’ı
olduğunu varsayarak, bir `List<Driver>`’ın bir `List<Person>` olduğunu düşünürüz. Aslında, geçen şey sürücü kayıtlarının
bir kopyasıdır. Aksi takdirde, nüfus bürosu listeye sürücü olmayan yeni kişiler ekleyebilir ve bu da DMV’nin kayıtlarını
bozabilir. Bu tür durumlarla başa çıkmak için daha esnek generic türleri düşünmek faydalıdır. Şimdiye kadar gördüğümüz
kurallar oldukça kısıtlayıcıdır.
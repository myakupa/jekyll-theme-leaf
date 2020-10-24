Merhabalar.  Bu yazımızda ![dvar]bu adresten ulaşabileceğiniz DVAR makinesinin çözümünü yapacağız. DVAR, ARM Linux tabanlı router sanal makinesidir. İçerisinde yönetici paneli görevini yapan bir web server çalıştırmaktadır. Bu web server'da bulunan zafiyeti kullanarak router'ı uzaktan kontrol edebilecek exploit kodu hazırlayacağız.
Sanal makineyi çalıştırdığımızda bize bir IP adresi veriyor ve bu adres ile router'a erişebiliyoruz. Alttaki resimde gözüktüğü gibi router üzerinde 3 farklı portta bazı servisler çalışmaktadır. 22 portu ile cihaza ssh bağlantısı yapabilir ve cihazı kontrol edebiliriz. Debugging aşamasında ssh bize yardımcı olacak. 80 ve 8080 portlarında iki farklı web server çalışmaktadır. 80 portunda bir router arayüzü bulunmaktadır. 8080 portunda ise bir trafik lambası  uygulaması bulunmaktır. Bu yazımızıda 80 portundaki servisi inceleyeceğiz. Trafik lambası uygulaması başka bir yazının konusudur.

<img src="{{site.baseurl}}/screenshot.png">



[dvar]: https://blog.exploitlab.net/2018/01/dvar-damn-vulnerable-arm-router.html

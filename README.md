# exchange-migration
Exchange 2013 to Exchange Online Hybrid Migration Step By Step

Tanım: Exchange 2013 On Premise altyapısında barınan hesapların (mail, arşiv, contacts, todo, calender) Microsoft 365 (Exchange Online) sistemine migrate edilmesi.

Önemli!!! Bu makale sadece bilgilendirme amaçlıdır. Lütfen migration işlemleri için microsoftun kendi sayfasındaki dökümanları referans alınız. İşlemlerden önce mevcut sunucunuzun sorunsuz bir yedeğinin olduğuna emin olunuz. Aşağıdaki işlemlerden dolayı meydana gelebilecek sorunlar sizin sorumluluğunuzdadır.

Bu aşamadan sonra lokal sunucu: on premise, remote sunucu exo olarak adlandırılacaktır.

Sabırsızlar için özet adımlar

- Microsoft 365 tarafında alan adınızı doğrulayın.
- Exchange on premise admin center üzerinde, hybrid menüsünden hybrid ayarlarınızı yapın ve on premise <-> exo arasındaki bağlantıyı sağlayın. (Ben SSL v.b. hatalarla uğraşmamak için full hybrid modu ile kurulum yaptım.)
- Hybrid kurulumunda sizden Active Directory Senkronizasyonu yapmak için Azure AD Connect yazılımı ile local AD üzerindeki kullanıcılarınızı Azure AD üzerine senkronize etmenizi isteyecek. Bu adımı dikkatlice yapıp, işlemin hatasız tamamlandığına emin olmadan diğer adımlara geçmeyin. Azure portal üzerinden kontrol ederek, tüm kullanıcılarınızın senkronize edildiğine emin olun.
- EXO Admin Center üzerinden, migration menüsüne gelin ve remote move seçeneği ile mail hesaplarını migrate etmeye başlayın. Migration end point yada migrate edilecek kullanıcılar karşınıza gelemzse bir yerde hatanız var demektir.
- Tüm migrate işlemleri bitince, MX kayıtlarınızı değiştirin ve AD üzerinde Directory Senkronizasyonunu kapatın. Böylece EXO üzerine aktardığınız tüm kullanıcılar cloud user a dönüşecek ve yönetebileceksiniz.
- Daha sonra on premise sunucunuz ile vedalaşabilirsiniz.

İşini garantiye alanlar için detay adımlar, karşılaşabileceğiniz hatalar ve çözümleri

Burada elimden geldiğince fazla detay vererek anlatmaya çalışacağım.

On Premise PowerShell bağlantısı: Windows üzerinde Exchange Management Shell'i bulup çalıştırın. Herhangi bir nedenle hata verir ve bağlanmazsa aşağıdaki komut ile bağlantı sağlayabilirsiniz.

connect-exchangeserver -serverfqdn mail.sunucuadi.com -username Administrator@domain.com

Hala bağlantı sağlayamıyor iseniz, PowerShell üzerinden bağlanmayı deneyin: https://learn.microsoft.com/en-us/powershell/exchange/connect-to-exchange-servers-using-remote-powershell?view=exchange-ps

Sağlıklı bir taşıma yapabilmek için, on premise exchange sunucusunun da sağlıklı olması gerekiyor. 

1) Exchange On Premise üzerinde Content Indexing sağlıklı durumda mı?
on premise'e exchange management shell ile bağlantı sağlayın
Get-MailboxDatabaseCopyStatus * | Sort Name | Select Name, Status, ContentIndexState
Bu komutun çıktısında ContentIndexState Healty yazması gerekmektedir.
Healty olanlar bir sonraki adıma geçebilir. Healty olmayanlar;
Stop-Service MSExchangeFastSearch
Stop-Service HostControllerService
Get-MailboxDatabase "DB01" | Select EdbFilePath -> burada DB01 yazan yere kendi exchange database adını yazın. Bu komutun çıktısı size database'in hangi klasörde barındırıldığını gösteriyor.
Örnek: D:\EXCHANGE-DB\DB01.edb

Şimdi D:\EXCHANGE-DB\ klasörüne gidin, orada harf ve rakamlardan oluşan, sonu .single ile biten uzun bir klasöre göreeceksiniz. Bu klasörün adını .single_eski şeklinde yeniden adlandırın.

Servisleri başlatın: Get-Service -Name "HostControllerService","MSExchangeFastSearch" | Start-Service

Yukarıdaki komut ile servisleri başlattıktan sonra, index yeniden oluşmaya başlayacak. Bu veri boyutuna göre bir kaç saat sürebilir. 

Aşağıdaki komutta indexing durumunu kontrol edebilirsiniz. Durum healty olduktan sonra 2. adıma geçebilirsiniz.
Get-MailboxDatabaseCopyStatus * | Sort Name | Select Name, Status, ContentIndexState

2) 

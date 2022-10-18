# exchange-migration
Exchange 2013 to Exchange Online Hybrid Migration Step By Step

Tanım: Exchange 2013 On Premise altyapısında barınan hesapların (mail, arşiv, contacts, todo, calender) Microsoft 365 (Exchange Online) sistemine migrate edilmesi.

Önemli!!! Bu makale sadece bilgilendirme amaçlıdır. Lütfen migration işlemleri için microsoftun kendi sayfasındaki dökümanları referans alınız. İşlemlerden önce mevcut sunucunuzun sorunsuz bir yedeğinin olduğuna emin olunuz. Aşağıdaki işlemlerden dolayı meydana gelebilecek sorunlar sizin sorumluluğunuzdadır. Microsoft, 2000 den daha az mail hesabı olan sistemler için hybrid migration önermiyor. Bunun yerine cutover migration öneriyor. Fakat ben posta hesaplarının içeriği çok büyük olduğu için hybrid migraiton tercih ettim.

Bu aşamadan sonra lokal sunucu: on premise, remote sunucu exo olarak adlandırılacaktır.

Hybrid nedir? Hybrid aslında bazı kullanıcıların On Premise üzerinde bazılarının ise EXO üzerinde aynı anda çalışabildiği ortak bir mimari.

Sabırsızlar için özet adımlar

- Microsoft 365 tarafında alan adınızı doğrulayın. **
- Exchange on premise admin center üzerinde, hybrid menüsünden hybrid ayarlarınızı yapın ve on premise <-> exo arasındaki bağlantıyı sağlayın. (Ben SSL v.b. hatalarla uğraşmamak için full hybrid modu ile kurulum yaptım.)
- Hybrid kurulumunda sizden Active Directory Senkronizasyonu yapmak için Azure AD Connect yazılımı ile local AD üzerindeki kullanıcılarınızı Azure AD üzerine senkronize etmenizi isteyecek. Bu adımı dikkatlice yapıp, işlemin hatasız tamamlandığına emin olmadan diğer adımlara geçmeyin. Azure portal üzerinden kontrol ederek, tüm kullanıcılarınızın senkronize edildiğine emin olun.
- EXO Admin Center üzerinden, migration menüsüne gelin ve remote move seçeneği ile mail hesaplarını migrate etmeye başlayın. Migration end point yada migrate edilecek kullanıcılar karşınıza gelemzse bir yerde hatanız var demektir.
- Tüm migrate işlemleri bitince, MX kayıtlarınızı değiştirin ve AD üzerinde Directory Senkronizasyonunu kapatın. Böylece EXO üzerine aktardığınız tüm kullanıcılar cloud user a dönüşecek ve yönetebileceksiniz.
- Daha sonra on premise sunucunuz ile vedalaşabilirsiniz.

İşini garantiye alanlar için detay adımlar, karşılaşabileceğiniz hatalar ve çözümleri

Burada elimden geldiğince fazla detay vererek anlatmaya çalışacağım.

On Premise PowerShell bağlantısı: Windows üzerinde Exchange Management Shell'i bulup çalıştırın. Herhangi bir nedenle hata verir ve bağlanmazsa aşağıdaki komut ile bağlantı sağlayabilirsiniz.

connect-exchangeserver -serverfqdn mail.domain.com -username Administrator@domain.com

Hala bağlantı sağlayamıyor iseniz, PowerShell üzerinden bağlanmayı deneyin: https://learn.microsoft.com/en-us/powershell/exchange/connect-to-exchange-servers-using-remote-powershell?view=exchange-ps

Sağlıklı bir taşıma yapabilmek için, on premise exchange sunucusunun da sağlıklı olması gerekiyor. 

1) Exchange On Premise tüm güncellemelerine sahip olduğunuza emin olun.

Get-ExchangeServer | Format-Table Name, Edition, AdminDisplayVersion komutu ile güncel versiyonu kontrol edebilirsiniz.

2) Exchange On Premise üzerinde 25 ve 443 portlarının firewallda açık olduğuna emin olun.

3) Exchange On Premise üzerinde Content Indexing sağlıklı durumda mı?

Eğer Content Indexing sağlılı değil ise, migration işlemi başlar ama asla sonlanmaz. Saatlerce boşuna beklersiniz.

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

Yukarıdaki komut ile servisleri başlattıktan sonra, index yeniden oluşmaya başlayacak. Bu işlem veri boyutuna göre bir kaç saat sürebilir. 

Aşağıdaki komutta indexing durumunu kontrol edebilirsiniz. Durum healty olduktan sonra 3. adıma geçebilirsiniz.
Get-MailboxDatabaseCopyStatus * | Sort Name | Select Name, Status, ContentIndexState

4) MRS Proxy yi aktif edin

Get-WebServicesVirtualDirectory | Format-Table Server,MRS*

Server    MRSProxyEnabled
------    ---------------
EXCH-2013           False

Get-WebServicesVirtualDirectory -ADPropertiesOnly | Where {$_.MRSProxyEnabled -ne $true} | Set-WebServicesVirtualDirectory -MRSProxyEnabled $true komutu ile enable edebilirsiniz.

Eğer GUI üzerinden aktif etmek isterseniz, on premise admin center -> servers -> virtual directories > edit -> general  "Enable MRS Proxy Endpoint" kutusunu işaretleyin.

Önemli: Burada external/internal URL bölümünde yazan URL lerinde geçerli bir SSL sertifikası olması gerekmektedir.

5) Hybrid configuration

Dilerseniz on premise admin center daki hybrid menüsünden ya da https://aka.ms/HybridWizard adreesini ziyaret ederek Hybrid Configuration Wizard'ı indirin ve yükleyin. Bu aşamada Exchange On Premise <-> Exchange Online arasındaki köprüyü kuruyor olacağız.
On Premise ve EXO login bilgilerinizi yazarak devam edin. 
Full Hybrid Configuration ve Organization Configuration Transfer seçeneklerini seçerek devam edin.
Exchange Classic Topology seçip kurulumu bitiriyoruz. 
Kurulum sonunda bir hata vermediyse işin %80'i bitti demektir.

6) Aktarılacak domainlerde SMTP alias var mı?

Örneğin siz user@domain.com için migration planlaması yapıyorsunuz. Fakat on premise üzerinde user@domain.com dışında user@domain.net şeklinde bir SMTP alias var ise 2 seçeneğiniz var, ya yeni domaini de microsoft admin center dan tanımlı alan adları listesine ekleyeceksiniz ya da on premise üzerinden bu aliasları sileceksiniz. Benim taşımamda bu aliaslar artık kullanılmadığı için silmeyi tercih ettim.

get-mailbox | select -expand emailaddresses alias komutu ile email aliaslarını görebilirsiniz.

On premise AD PowerShell de Start-ADSyncSyncCycle -PolicyType Delta komutunu çalıştırın ki bu değişiklikler azure ad üzerinde senkronize olsun. Ya da 20-30 dakika beklerseniz kendi kendine de senkronize olacaktır.

Bu aşamadan sonra exchange on premise üzerinde pek bir işlem kalmadı.

7) EXO üzerine powercli ile bağlanma

PowerShell ile connect-exo.ps1 dosyasını çalıştırın ve exo kullanıcı adı ve şifrenizi girin.
Get-MigrationEndpoint | Format-List Identity, RemoteServer komutu ile hybrid configuration ayarlarınızı teyit edin. 

Aşağıdaki gibi bir çıktı aldıysanız, köprü kurulmuş demektir.

Identity     : Hybrid Migration Endpoint - EWS (Default Web Site)
RemoteServer : 7436112b-b224-4f99-8134-ba8ea4033946.resource.mailboxmigration.his.msappproxy.net

Identity     : mail.domain.com
RemoteServer : mail.domain.com

7) Migrate işlemlerini 2 şekilde başlatabilirsiniz. 

Ya EXO Admin Center üzerinden ya da EXO'ye powershell ile bağlanıp komut satırı ile. 
Ben ikisine de örnek vereceğim;

EXO admin center:

- Migration Path
Give migration batch a unique name: migration job için bir isim girin
Select the mailbox migration path: Migration ton Exchange Online (EXO)

- Migration Type
Remote Move Migration

- Set a migration endpoint: Hybrid Migration Endpoint - EWS (Default Web Site)

- Add users
Manually add users to migrate: Burada en küçük posta kutusu boyutuna sahip olan hesap ile denemelere başlayabilirsiniz. Benim önerim önce boş/yeni bir posta hesabı açıp onunla deneme yapmanız. Her şey yolunda ise gerçek kullanıcılarla devam edebiliriniz. Basitçe on premise üzerinde Get-Mailbox komutu ile mailbox boyutlarını görebilirsiniz. Eğer çoklu taşıma yapmak isterseniz csv dosyasına mail hesaplarını yazarak aynı anda birden fazla aktarım yapabilirsiniz. Fakat hesap çok fazla değil ise ben tek tek yapma taraftarıyım. Çünkü bir anda on premise sunucusuna yüklenirseniz, disk yavaşladığı için zaten işlemleri ya yavaş yapıyor ya da bazı işlemleri kuyrukta bekletiyor. Benim gördüğüm 200mbit/sn ile taşımak ideal.

- Configuration
Target delivery domain: domain.mail.onmicrosoft.com

- Schedule batch migration
Burada işlemi hemen başlatabilir ya da istediğiniz bir tarih/saat için planlayabilirsiniz.

EXO Power Shell üzerinden migration başlatmak isterseniz:

New-MoveRequest -Identity "user@domain.com.tr" -Remote -RemoteHostName "7436112b-b224-4f99-8134-ba8ea4033946.resource.mailboxmigration.his.msappproxy.net" -TargetDeliveryDomain "domain_adi.mail.onmicrosoft.com" -RemoteCredential (Get-Credential Administrator@domain.com)

Not: Powershell ile başlattığınız migration işlemleri admin panelde gözükmez. Sadece powershell komutları ile migration durumunu takip edebilirsiniz.

8) Migrate işlemlerinin durumunun takip edilmesi

Burada iki farklı komut ile migration durumunu takip edebilirsiniz.

Get-MoveRequest | Get-MoveRequestStatistics -> Bu işlemi % olarak gösteriyor.
Get-MigrationUser | Get-MigrationUserStatistics -> Bu işlemi item bazında gösteriyor.

Bonus:

Get-MoveRequest –BatchName "burak" | Get-MoveRequestStatistics | ft DisplayName,StatusDetail,DataConsistencyScore,PercentComplete burada migration işlemine burak ismini verdiyseniz, sadece o kullanıcının migration durumunu görebilirsiniz.
Get-MigrationUser | Get-MigrationUserStatistics | Where-Object {$_.Status -notlike "Completed"} burada durumu devam edenleri görebilirsiniz. PowerShell komutlarının sonu yok, bir çok varyasyona araştırarak ulaşabilirsiniz.

9) Migration işleminin tamamlanması.

Migration işlemi completed olduğunda, örneğin mailbox üzerinde 200k item var ise, bazen 3-4 item aktarılamıyor. Bu nedenme migration status approve skipped items'e düşüyor. Bu migrasyon işlemini manual tanımlamak için exo powershell de Set-MigrationUser -ApproveSkippedItems komutu ve arından Identity sorduğunda email adresini yazarak taşımayı tamamlayabiliriz.

Taşıma tamamlandıktan sonra, Exchange On Premise üzerinde bu kullanıcı artık O365 olarak gözükecek.

10) Tüm kullanıcılara Microsoft Admin Center üzerinden lisanslarını tanımlayın. 
11) EXO üzerinde Directory Synchronization kapatarak, tüm kullanıcıları cloud user'a çeviriyoruz.

EXO Power Shell üzerinde: Set-MsolDirSyncEnabled -EnableDirSync $false

12) Exchange On Premise devre dışı bırakılması: https://learn.microsoft.com/en-us/exchange/decommission-on-premises-exchange

Sorular:

1) MX kayıtlarını ne zaman değiştirmeliyim? **

Diğer firmalar nasıl yapıyor bilemiyorum fakat domain doğrulamada şöyle bir sorun yaşadık. EXO tarafında domaini eklemek için MX kaydının EXO ya yönlendirilmesini istiyordu. Fakat henüz on premise <-> exo arasındaki köprüyü kurmamıştık. Bu nedenle gece mail trafiğinin çok az olduğu bir saatte kısa süreli MX leri EXO ya çevirip domaini doğrulattık. Sonra MX leri eski haline aldık. Daha sonra isterseniz hybrid configuration yaptıktan sonra ya da tüm migration süreçleri bittikten sonra MX leri çevirebilirsiniz.

2) Hem Exchange On Premise hem de EXO üzerinde mail trafiği nasıl çalışıyor?

Hybrid configuration yapıldığında, hem on premise hem de exo tarafında Mail Flow -> Connectors bölümünde inbound ve outbound connectorler görecelsiniz. Bunlar geçiş sürecinde mail trafiğini yönetiyorlar.





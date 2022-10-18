# exchange-migration
Exchange 2013 to Exchange Online Hybrid Migration Step By Step

Tanım: Exchange 2013 on premise altyapısında barınan eposta ve arşivlerin Microsoft 365 (Exchange Online) sistemine migrate edilmesi.

Bu aşamadan sonra lokal sunucu: on premise, remote sunucu exo olarak adlandırılacaktır.

Sabbırsızlar için özet adımlar

- Microsoft 365 tarafında alan adınızı doğrulayın.
- Exchange on premise admin center üzerinde, hybrid menüsünden hybrid ayarlarınızı yapın ve on premise <-> exo arasındaki bağlantıyı sağlayın. (Ben SSL v.b. hatalarla uğraşmamak için full hybrid modu ile kurulum yaptım.)
- Hybrid kurulumunda sizden Active Directory Senkronizasyonu yapmak için Azure AD Connect yazılımı ile local AD üzerindeki kullanıcılarınızı Azure AD üzerine senkronize etmenizi isteyecek. Bu adımı dikkatlice yapıp, işlemin hatasız tamamlandığına emin olmadan diğer adımlara geçmeyin. Azure portal üzerinden kontrol ederek, tüm kullanıcılarınızın senkronize edildiğine emin olun.
- EXO Admin Center üzerinden, migration menüsüne gelin ve remote move seçeneği ile mail hesaplarını migrate etmeye başlayın. 
- Tüm migrate işlemleri bitince, MX kayıtlarınızı değiştirin ve AD üzerinde Directory Senkronizasyonunu kapatın. Böylece EXO üzerine aktardığınız tüm kullanıcılar cloud user a dönüşecek ve yönetebileceksiniz.
- Daha sonra on premise sunucunuz ile vedalaşabilirsiniz.

İşini garantiye alanlar için detay adımlar, karşılaşabileceğiniz hatalar ve çözümleri


# Kuyruklar

- [Yapılandırma](#configuration)
- [Temel Kullanım](#basic-usage)
- [Kuyruğa Closure Fonksiyonu Ekleme](#queueing-closures)
- [Kuyruk Dinleyicileri Çalıştırma](#running-the-queue-listener)
- [Daemon Kuyruk İşçisi](#daemon-queue-worker)
- [Push Kuyrukları](#push-queues)
- [Başarısız İşler](#failed-jobs)

<a name="configuration"></a>
## Yapılandırma

Laravel'in Queue (kuyruk) bileşeni bir takım farklı kuyruk servisleri için tek bir API sağlamaktadır. Kuyruklar e-mail göndermek gibi zaman harcayan görevleri ileri bir zamana kadar ertelemenize imkan verir ve böylece uygulamanıza yapılan web istekleri büyük ölçüde hızlanır.

Kuyruk yapılandırma dosyası `app/config/queue.php` olarak saklanır. Bu dosyada framework'e dahil edilmiş kuyruk sürücülerinin her birisi için bağlantı yapılandırmaları bulacaksınız. Laravel'deki kuyruk sürücüleri arasında [Beanstalkd](http://kr.github.com/beanstalkd), [IronMQ](http://iron.io), [Amazon SQS](http://aws.amazon.com/sqs), [Redis](http://redis.io) ve senkronize (lokal kullanım için) sürücü yer almaktadır.

Listelenen bu kuyruk sürücüleri için aşağıdaki bağımlılıklar gereklidir:

- Beanstalkd: `pda/pheanstalk`
- Amazon SQS: `aws/aws-sdk-php`
- IronMQ: `iron-io/iron_mq`

<a name="basic-usage"></a>
## Temel Kullanım

#### Bir İşin Kuyruğa Sokulması

Kuyruğa yeni bir iş itmek için `Queue::push` metodunu kullanın:

	Queue::push('SendEmail', array('message' => $message));

#### Bir İş İşleyicisinin Tanımlanması

`push` metoduna girilen ilk parametre işi yapmak için kullanılacak sınıfın adıdır. İkinci parametre işleyiciye geçirilecek veri dizisidir. Bir iş işleyicisi şu şekilde tanımlanmalıdır:

	class SendEmail {

		public function fire($is, $veri)
		{
			//
		}

	}

Gerekli olan tek metodun `fire` olduğuna dikkat edin. Bu metod bir `iş` olgusu ve bir de kuyruğa sokulacak `veri` dizisi parametrelerini alır.

#### Özel Bir İşleyici Metodunun Belirlenmesi

Eğer iş'in `fire`'den başka bir metod kullanmasını istiyorsanız, işi sokarken (yani push metodunda) metodu belirleyebilirsiniz:

	Queue::push('SendEmail@send', array('message' => $message));

#### Bir İş İçin Kuyruk / Tüp Belirtilmesi

Bir işin gönderilmesi gereken kuyruğu / tüpü de belirtebilirsiniz:

	Queue::push('SendEmail@send', array('message' => $message), 'emails');

#### Birden Çok İş İçin Aynı Yükün Geçilmesi

Birkaç kuyruk işi için aynı veriyi geçmeniz gerekiyorsa, `Queue::bulk` metodunu kullanabilirsiniz:

	Queue::bulk(array('SendEmail', 'NotifyUser'), $payload);

#### Bir İşin Çalıştırılmasının Geciktirilmesi

Kimi zaman sıraya sokulmuş bir işin çalıştırılmasını geciktirmek isteyebilirsiniz. Örneğin, bir müşteriye kayıt olduktan 15 dakika sonra bir e-posta gönderen bir işi kuyruğa koymak isteyebilirsiniz. Bunu `Queue::later` metodunu kullanarak başarabilirsiniz:

	$date = Carbon::now()->addMinutes(15);

	Queue::later($date, 'SendEmail@send', array('message' => $message));

Bu örnekte, işe atamak istediğimiz gecikme süresini belirtmek için [Carbon](https://github.com/briannesbitt/Carbon) date kitaplığını kullanıyoruz. Alternatif olarak geciktirmek istediğiniz saniye sayısını tam sayı olarak geçebilirsiniz.

#### İşlenmiş Bir İşin Silinmesi

Bir iş işlendikten sonra kuyruktan silinmelidir. Silme işlemi ilgili `iş` olgusunda `delete` metodu kullanılarak yapılabilir:

	public function fire($is, $veri)
	{
		// İşi işle...

		$is->delete();
	}

#### Bir İşin Tekrar Kuyruğa Koyulması

Bir işi tekrar kuyruğa devretmek isterseniz, bunu `release` metodu aracılığıyla yapabilirsiniz:

	public function fire($is, $veri)
	{
		// İş sürecini yürüt...

		$is->release();
	}

İş tekrar salınmadan önce kaç saniye bekleneceğini de belirleyebilirsiniz:

	$is->release(5);

#### Çalıştırma Girişimlerinin Sayısını Yoklama

İş işlenirken bir istisna oluşursa, otomatik olarak kuyruğa tekrar salınacaktır. `attempts` metodunu kullanarak, işi çalıştırmak için yapılmış olan girişim sayısını da yoklayabilirsiniz:

	if ($is->attempts() > 3)
	{
		//
	}

#### Bir İşin ID'sine Erişme

İş tanımlayıcılarına da erişebilirsiniz:

	$is->getJobId();

<a name="queueing-closures"></a>
## Kuyruğa Closure Fonksiyonu Ekleme

Kuyruğa bir Closure de push edebilirsiniz. Bu, kuyruğa sokulması gerekecek hızlı, basit görevler için çok uygundur:

#### Kuyruğa Bir Closure Sokulması

	Queue::push(function($is) use ($id)
	{
		Account::delete($id);

		$is->delete();
	});

> **Not:** Kuyruğa sokulmuş Closure'lar için nesneleri `use` direktifi aracılığıyla kullanılabilir yapmak yerine, birincil anahtarları geçmeyi ve ilgili modeli kuyruk işiniz içinden tekrar çekmeyi düşünün. Bu, beklenmedik serileştirme davranışlarını çoğu keresinde önleyecektir.

Iron.io [push kuyrukları](#push-queues) kullanılıyorken, Closure'ların kuyruğa sokulmasında daha fazla önlem almalısınız. Kuyruk mesajlarızı alan son nokta, isteğin gerçekten Iron.io'den mi geldiğini doğrulayacak bir jeton yoklaması yapmalıdır. Örneğin, sizin push kuyruk son noktanız şuna benzer bir şey olmalıdır: `https://uygulamaniz.com/queue/receive?token=SecretToken`. Böylece, kuyruk istek sıralamasından önce uygulamanızdaki gizli jetonun değerini kontrol edebilirsiniz.

<a name="running-the-queue-listener"></a>
## Kuyruk Dinleyicileri Çalıştırma

Laravel, kuyruğa itildikçe yeni işler çalıştıran bir Artisan görevi içermektedir. Bu görevi çalıştırmak için `queue:listen` komutunu kullanabilirsiniz:

#### Kuyruk Dinleyici Başlatılması

	php artisan queue:listen

Ayrıca dinleyicinin kullanacağı kuyruk bağlantısını da belirtebilirsiniz:

	php artisan queue:listen connection

Unutmamanız gereken şey, bu görev başlatıldıktan sonra elle durdurulana kadar çalışmaya devam edeceğidir. Kuyruk dinleyicinin çalışmayı durdurmamasından emin olmak için [Supervisor](http://supervisord.org/) gibi bir süreç monitörü kullanabilirsiniz.

Kuyruk önceliklerini ayarlamak için `listen` komutuna virgülle ayrılmış bir kuyruk bağlantıları listesi geçebilirsiniz:

	php artisan queue:listen --queue=high,low

Bu örnekte, `high-connection` üzerindeki işler, her zaman için `low-connection`'dan gelen işlere geçmeden önce yürütülecektir.

#### İş Zaman Aşımı Parametresi Belirleme

Ayrıca her işin çalışmasına izin verilecek zaman süresini (saniye cinsinden) de ayarlayabilirsiniz:

	php artisan queue:listen --timeout=60

#### Kuyruk Uyku Süresinin Belirtilmesi

Ek olarak, yeni işin eyleme alınmadan önce beklenileceği süreyi saniye cinsinden belirtebilirsiniz:

	php artisan queue:listen --sleep=5

Not: kuyruk sadece kuyrukta iş olmadığı takdirde "uyur". Eğer kuyrukta başka işler varsa, kuyruk uyumaksızın onları çalışmaya devam edecektir.

#### Kuyruktaki İlk İşin İşleme Geçirilmesi

Kuyruktaki sadece ilk sıradaki işi yürütmek için `queue:work` komutunu kullanabilirsiniz:

	php artisan queue:work

<a name="daemon-queue-worker"></a>
## Daemon Kuyruk İşçisi

`queue:work` ayrıca işlerin işlenmesinin framework tekrar boot edilmeksizin devam etmesi için kuyruk işçisinin zorlanması için bir `--daemon` seçeneği içermektedir. Bu, `queue:listen` komutuyla karşılaştırıldığında CPU kullanımında önemli bir azalmayla sonuçlanır ama yayımlama sırasında halihazırda çalışmakta olan kuyrukların drene edilmesi gerekliliği karmaşıklığını ekler.

Bir kuyruk işçisini daemon modunda başlatmak için, `--daemon` flagını kullanın:

	php artisan queue:work connection --daemon

	php artisan queue:work connection --daemon --sleep=3

	php artisan queue:work connection --daemon --sleep=3 --tries=3

Gördüğünüz gibi, `queue:work` komutu `queue:listen` için kullanılan seçeneklerin pek çoğunu desteklemektedir. Mevcut seçeneklerin tümünü görmek için `php artisan help queue:work` komutunu kullanabilirsiniz.

### Daemon Kuyruk İşçileriyle Yayımlama

Bir uygulamayı daemon kuyruk işçileri kullanarak yayımlamanın en basit yolu yayımlamanızın en başında uygulamanızı bakım (maintenance) moduna koymaktır. Bu `php artisan down` komutu kullanılarak yapılabilir. Uygulama bakım moduna alındıktan sonra, Laravel artık kuyruğa yeni işler kabul edecektir ama mevcut işleri işlemeye devam edecektir. Mevcut işlerinizin hepsinin çalışması için yeterli zaman (genellikle 30-60 saniyeden daha uzun değildir) geçtikten sonra, işçiyi durdurabilir ve yayımlama sürecinize devam edebilirsiniz.

Şayet Supervisor veya Laravel Forge (Supervisor kullanır) kullanıyorsanız, tipik olarak bir işçiyi aşağıdakine benzer bir komutla durdurabilirsiniz:

	supervisorctl stop worker-1

Kuyruklar drene edildikten ve yeni kodunuz sunucunuza yayımlandıktan sonra, daemon kuyruk işçisini yeniden başlatmalısınız. Eğer Supervisor kullanıyorsanız, bu tipik olarak şuna benzer bir komutla yapılabilir:

	supervisorctl start worker-1

<a name="push-queues"></a>
## Push Kuyrukları

Push kuyrukları size herhangi bir art alan veya arka plan dinleyici çalıştırmaksızın güçlü Laravel 4 kuyruk araçlarını kullanmanıza imkan verir. Push kuyrukları şu anda sadece [Iron.io](http://iron.io) sürücüsü tarafından desteklenmektedir. Başlamak için önce bir Iron.io hesabı oluşturun ve Iron kimlik bilgilerinizi `app/config/queue.php` yapılandırma dosyasına ekleyin.

#### Bir Push Kuyruk Aboneliğinin Kayda Geçirilmesi

Daha sonra, yeni push edilmiş kuyruk işlerini alacak bir URL son noktasını kayda geçirmek için `queue:subscribe` Artisan komutunu kullanabilirsiniz:

	php artisan queue:subscribe queue_name http://falan.com/queue/receive

Şimdi, sizin Iron panonuza giriş yaptığınız zaman, yeni push kuyruğunuzu ve abone olunan URL'yi göreceksiniz. Verilen bir kuyruk için istediğiniz kadar çok URL kaydedebilirsiniz. Sonra da, `queue/receive` son noktanız için bir rota oluşturun ve `Queue::marshal` metodundan cevap döndürün:

	Route::post('queue/receive', function()
	{
		return Queue::marshal();
	});

Doğru iş işleyici sınıfının ateşlenmesiyle `marshal` metodu ilgilenecektir. Push kuyruğundaki işleri ateşlemek için, konvansiyonal kuyruklar için kullanılan aynı `Queue::push` metodunu kullanmanız yeterlidir.

<a name="failed-jobs"></a>
## Başarısız İşler

İşler her zaman planladığımız şekilde gitmediğinden, bazen kuyruğa soktuğumuz işler başarılamaz. Dert etmeyin, bu en iyilerimizin bile başına gelir! Laravel bir işin en fazla kaç defa denenmesi gerektiğini belirtmeniz için kolay bir yol içerir. Bir iş bu girişme miktarını aştıktan sonra, `failed_jobs` (başarısız işler) tablosuna eklenecektir. Başarısız işler tablosunun adını `app/config/queue.php` yapılandırma dosyasında ayarlayabilirsiniz.

`failed_jobs` tablosu için bir migrasyon oluşturmak için, `queue:failed-table` komutunu kullanabilirsiniz:

	php artisan queue:failed-table

Bir işin maksimum kaç defa yapılma girişiminde bulunulacağını `queue:listen` komutunda `--tries` anahtarını kullanarak belirtebilirsiniz:

	php artisan queue:listen connection-name --tries=3

Eğer bir kuyruk işi başarısız olduğu takdirde çağrılacak bir olay kayda geçirmek isterseniz, `Queue::failing` metodunu kullanabilirsiniz. Bu olay ekibinizi bir e-posta veya [HipChat](https://www.hipchat.com) aracılığıyla bilgilendirmek için harika bir fırsattır.

	Queue::failing(function($connection, $job, $data)
	{
		//
	});

Başarısız olmuş işlerinizin tümünü görmek için `queue:failed` Artisan komutunu kullanabilirsiniz:

	php artisan queue:failed

Bu `queue:failed` komutu iş ID, bağlantı, kuyruk ve başarısızlık zamanını listeleyecektir. Bunlardan iş ID başarısız işi yeniden denemek için kullanılabilir. Örneğin, ID'si 5 olan başarısız bir işi yeniden denemek için aşağıdaki komut verilmelidir:

	php artisan queue:retry 5

Başarısız bir işi silmek isterseniz, `queue:forget` komutunu kullanabilirsiniz:

	php artisan queue:forget 5

Başarısız işlerinizin tümünü silmek için `queue:flush` komutunu kullanabilirsiniz:

	php artisan queue:flush
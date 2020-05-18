---
layout: post
title:  "Python ile SSH Brute Force İnceleme"
author: fatih
categories: [ Python, Siber Güvenlik ]
image: assets/images/ssh.png
tags: [Python]
---

Geçen günlerde bir sunucumuzun aldığı SSH Brute Force saldırısında log kayıtlarına baktığımızda 80000+ fazla deneme yapıldığını gördük. Bu yazıda saldırının geldiği ip adreslerinin hangi ülkelere kayıtlı olduğunu tespit etmek için yazdığım python kodunu paylaşacağım.

![SSH Brute Force]({{ site.baseurl }}/assets/images/ssh1.JPG)

Öncelikle basit bir bash komutu ile SSH hatalı giriş denemelerini bir dosyaya yazalım.
`grep sshd.\*Failed /var/log/auth.log > /root/fzr/log.txt`

Sonra ```log.txt``` dosyası ile aynı konumda bir python dosyası oluşturalım ve düzenleyelim.
`touch crawl.py`
`nano crawl.py`

Python dosyasına ilk olarak kütüphaneleri ve gerekecek değişkenleri tanımlıyorum.
{% highlight python %}
from requests import get
file1 = open('log.txt', 'r') 
Lines = file1.readlines() 
ip_list = []
site = 0
{% endhighlight %}

Bu aşamadan sonra yapmamız gereken log kayıtlarındaki ip adreslerini ```ip_list``` değişkeninde toplamak olacaktır. Ip adreslerini ```ip_list``` değişkenine eklerken tekrarlama olmaması için iki farklı yol var.

### Yol 1
{% highlight python %}
for line in Lines:
    ip = line[line.find('from ')+len('from '):line.find(' port')]
    if not ip in ip_list:
        ip_list.append(ip)
{% endhighlight %}

### Yol 2
{% highlight python %}
for line in Lines:
    ip_list.append(line[line.find('from ')+len('from '):line.find(' port')])
ip_list = list(set(ip_list))
{% endhighlight %}

Ip adreslerini ayrıştırıp toparladığımıza göre artık konumlarını belirleyebiliriz. Konum belirlerken kullanacağımız uygulamalardan ilki 1000 sorguya kadar üyelik gerektirmeyen [ipapi](https://ipapi.co/) ve ikincisi [ipgeolocation](https://ipgeolocation.io/). Bu iki uygulamaya API araclığı ile ip adreslerini gönderip sonuçları metin belgesine yazdıracağız. İki uygulama tercih etmemin sebebi uygulamalardaki günlük sınırlamalar. Ayrıcı bu iki uygulamanın JSON cevap yapısı birbirine benziyor böylece ikisinden de gelecek veriyi kolayca okuyabileceğiz. Uygulamalara istek gönderirken ```requests``` kütüphanesinin ```get()``` fonksiyonunu kullanacağız. Ip adresleri 1700 taneden fazla olduğu için iki uygulamaya istekleri sıra sıra göndereceğiz.

{% highlight python %}
for ip in ip_list:
    try:
        if site == 0:
            loc = get('https://ipapi.co/'+ip+'/json/')
            site = 1
        elif site == 1:
            loc = get('https://api.ipgeolocation.io/ipgeo?apiKey=36c226c307da40b78a4629a26442e54d&ip='+ip)
            site = 0
        with open('result.txt', 'a') as result_file:
            result_file.write(ip + "==>" + loc.json()['city'] + "/" + loc.json()['country_name'] + "\n")
    except:
        continue
{% endhighlight %}

Kodumuzun son parçasınıda yazdıktan sonra artık çalıştırıp verileri inceleyebiliriz.
![SSH Brute Force]({{ site.baseurl }}/assets/images/ssh1.JPG)

## Kodun Tamamı
{% highlight python %}
from requests import get
file1 = open('log.txt', 'r') 
Lines = file1.readlines() 
ip_list = []
site = 0

for line in Lines:
    ip_list.append(line[line.find('from ')+len('from '):line.find(' port')])
ip_list = list(set(ip_list))

for ip in ip_list:
    try:
        if site == 0:
            loc = get('https://ipapi.co/'+ip+'/json/')
            site = 1
        elif site == 1:
            loc = get('https://api.ipgeolocation.io/ipgeo?apiKey=36c226c307da40b78a4629a26442e54d&ip='+ip)
            site = 0
        with open('result.txt', 'a') as result_file:
            result_file.write(ip + "==>" + loc.json()['city'] + "/" + loc.json()['country_name'] + "\n")
    except:
        continue
{% endhighlight %}
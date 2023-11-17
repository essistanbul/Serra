import os
import requests# HTTP istekleri göndermek ve yanıtları almak için kullanılır.
from bs4 import BeautifulSoup
from PIL import Image
from io import BytesIO
import sqlite3

def create_table_if_not_exists():
    conn = sqlite3.connect('resimler.sqlite3')
    cursor = conn.cursor() #nesnesi, veritabanı üzerinde sorguları çalıştırmak ve sonuçlarını almak için kullanılır.
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS resimler (
            dosya_adi TEXT,
            genislik INTEGER,
            yukseklik INTEGER,
            dosya_yolu TEXT,
            url TEXT
        )
    ''')
    conn.commit()
    conn.close()

def insert_image_data(dosya_adi, genislik, yukseklik, dosya_yolu, url):
    conn = sqlite3.connect('resimler.sqlite3')
    cursor = conn.cursor()
    cursor.execute('INSERT INTO resimler (dosya_adi, genislik, yukseklik, dosya_yolu, url) VALUES (?, ?, ?, ?, ?)',
                   (dosya_adi, genislik, yukseklik, dosya_yolu, url))
    conn.commit()
    conn.close()

def download_images_with_size_criteria(search_term, limit, min_width, min_height, max_width, max_height):
    print(f"{search_term} terimiyle ilgili resimler indiriliyor... Lütfen bekleyin.")
    
    # Klasörü oluştur veya kontrol et
    if not os.path.exists("indirilen_resimler"):
        os.makedirs("indirilen_resimler") # dizinin yolunu oluşturma
    
    query = search_term.replace(" ", "+")#Arama terimi, Google Görseller'deki URL için doğru formata dönüştürülür.

    url = f"https://www.google.com.tr/search?q={query}&tbm=isch"

    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36"
    }

    response = requests.get(url, headers=headers)#Belirtilen URL'ye HTTP GET isteği gönderilir ve yanıt alınır
    soup = BeautifulSoup(response.text, 'html.parser')# HTML içeriğine ayrıştırılır,web tarayıcı için kullanılır.

    image_links = []#boş bir liste oluşturma
    for img in soup.find_all('img'):#Html belgesindeki tüm resimleri img liste halinde döndürme
        src = img.get('src')
        if src:#src kaynak kodu bulmak
            image_links.append(src)#resimleri tek bir öğede ekleme

    downloaded_count = 0
    create_table_if_not_exists()#belirli tablo oluşturmak
    
    for link in image_links:# listesinde bulunan her resim bağlantısı için işlem yapar.
        try:
            response = requests.get(link, headers=headers) 
            if response.status_code == 200:#response nesnesinin status_code özelliğini kontrol ederek, isteğin başarılı bir şekilde tamamlandığını (HTTP durum kodu 200) doğrulama
                img = Image.open(BytesIO(response.content)) # PİL resim nesnesine dönüşme
                if min_width <= img.width <= max_width and min_height <= img.height <= max_height:
                    dosya_adi = f"{search_term}_{downloaded_count}.jpg"
                    dosya_yolu = os.path.join("indirilen_resimler", dosya_adi)
                    img.save(dosya_yolu)
                    insert_image_data(dosya_adi, img.width, img.height, dosya_yolu, link) # resim bilgilerini veri tabanına kayıt etmek
                    downloaded_count += 1 #downloaded_count değişkeni artırılarak kaç resim indirildiği takip edilir.
                    print("URL:", link)  # URL'yi terminale yazdır
                    if downloaded_count >= limit: #belirtilen bir sınıra (limit) ulaştığında veya bu sınıra eşit veya daha fazla olduğunda bir koşulu kontrol eder.
                        break # belirli bir koşulun karşılandığında döngüyü sonlandırmak veya belirli bir işlemi durdurmak için kullanılır. 
        except Exception as e:
            print(f"Hata oluştu: {e}")

    print(f"{downloaded_count} adet resim indirildi ve veritabanına kaydedildi.")


name = input("Lütfen arama terimini giriniz: ")
limit = int(input("İndirilecek resim limitini giriniz: "))
timeout = int(input("İstek zaman aşımını giriniz (saniye cinsinden): "))
width = int(input("Minimum genişlik değerini giriniz: "))
height = int(input("Minimum yükseklik değerini giriniz: "))
max_width = int(input("Max genişlik değerini giriniz: "))
max_height = int(input("Max yükseklik değerini giriniz: "))
download_images_with_size_criteria(name, limit, width, height, max_width, max_height) 


# Google Maps Rota Planlayıcı

Bu proje, Vue.js ve Tailwind CSS kullanarak oluşturulmuş bir Google Maps uygulamasıdır. Kullanıcılar haritada noktalar seçebilir, bu noktalar arasındaki mesafeyi, rakımı ve diğer bilgileri görebilirler.

## Özellikler

- Google Maps entegrasyonu
- Haritada nokta işaretleme ve sürükleme
- Noktalar arasında rota çizimi
- Toplam mesafe hesaplama
- Yükseklik verisi gösterimi
- Noktalar arası engel tespiti (dağlar, binalar vb.)
- Responsive tasarım

## Başlarken

### Gereksinimler

- Node.js (v14 veya üstü)
- npm veya yarn
- Google Maps API Anahtarı

### Kurulum

1. Projeyi klonlayın:
   ```bash
   git clone [repo-url]
   cd google-maps-project
   ```

2. Bağımlılıkları yükleyin:
   ```bash
   npm install
   # veya
   yarn install
   ```

3. Google Maps API Anahtarı Alın:
   - [Google Cloud Console](https://console.cloud.google.com/)'a gidin
   - Yeni bir proje oluşturun
   - Maps JavaScript API'yi etkinleştirin
   - API anahtarınızı alın

4. API Anahtarınızı Ekleyin:
   - `src/components/GoogleMap.vue` dosyasında `API_KEY` değişkenini kendi API anahtarınızla değiştirin.

5. Geliştirme sunucusunu başlatın:
   ```bash
   npm run dev
   # veya
   yarn dev
   ```

6. Tarayıcınızda `http://localhost:5173` adresine gidin.

## Kullanım

- Haritada bir nokta işaretlemek için haritaya tıklayın.
- İşaretçileri sürükleyerek konumlarını değiştirebilirsiniz.
- Bir işaretçiye tıkladığınızda detaylı bilgi görüntülenir ve kaldırma seçeneği sunulur.
- İki veya daha fazla nokta işaretlendiğinde, aralarında bir çizgi oluşturulur ve mesafe hesaplanır.
- Tüm noktaları temizlemek için "Tüm Noktaları Temizle" düğmesini kullanabilirsiniz.

## Teknolojiler

- [Vue.js](https://vuejs.org/)
- [Tailwind CSS](https://tailwindcss.com/)
- [Google Maps JavaScript API](https://developers.google.com/maps/documentation/javascript/overview)
- [Vite](https://vitejs.dev/)

## Lisans

Bu proje MIT lisansı altında lisanslanmıştır. Daha fazla bilgi için [LICENSE](LICENSE) dosyasına bakın.

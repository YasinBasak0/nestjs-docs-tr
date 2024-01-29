### Şifreleme ve Hashleme

**Şifreleme**, bilgileri kodlama sürecidir. Bu süreç, bilinen adıyla açık metin olan bilginin orijinal temsilini, şifreleme olarak bilinen alternatif bir forma, yani şifreleme yapar. İdeal olarak, sadece yetkili kişiler, şifreli bir metni şifre çözme ve orijinal bilgiye erişme yeteneğine sahip olmalıdır. Şifreleme, başlı başına müdahaleyi önlemez ancak anlamlı içeriği bir müdahaleciye bırakmaz. Şifreleme, iki yönlü bir işlevdir; şifrelenen şey, uygun anahtarla şifresi çözülebilir.

**Hashleme**, belirli bir anahtarı başka bir değere dönüştürme sürecidir. Bir hash fonksiyonu, matematiksel bir algoritma doğrultusunda yeni değeri oluşturmak için kullanılır. Hashleme yapıldıktan sonra çıkıştan girişe gitmek imkansız olmalıdır.

#### Şifreleme

Node.js, string'leri, sayıları, tamponları, akışları ve daha fazlasını şifrelemek ve şifresini çözmek için kullanabileceğiniz yerleşik bir [crypto modülü](https://nodejs.org/api/crypto.html) sağlar. Nest kendisi, gereksiz soyutlamaları önlemek için bu modülün üzerine ek bir paket sağlamaz.

Örneğin, AES (Advanced Encryption System) `'aes-256-ctr'` algoritması CTR şifreleme modunu kullanalım.

```typescript
import { createCipheriv, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const iv = randomBytes(16);
const password = 'Anahtar oluşturmak için kullanılan şifre';

// Anahtar uzunluğu algoritmaya bağlıdır.
// Bu durumda aes256 için 32 byte'tır.
const key = (await promisify(scrypt)(password, 'tuz', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);

const şifrelenecekMetin = 'Nest';
const şifrelenmişMetin = Buffer.concat([
  cipher.update(şifrelenecekMetin),
  cipher.final(),
]);
```

Şimdi `şifrelenmişMetin` değerini çözmek için:

```typescript
import { createDecipheriv } from 'crypto';

const çözücü = createDecipheriv('aes-256-ctr', key, iv);
const çözülenMetin = Buffer.concat([
  çözücü.update(şifrelenmişMetin),
  çözücü.final(),
]);
```

#### Hashleme

Hashleme için, [bcrypt](https://www.npmjs.com/package/bcrypt) veya [argon2](https://www.npmjs.com/package/argon2) paketlerini kullanmanızı öneririz. Nest kendisi, öğrenme eğrisini kısa tutmak için bu modüllerin üzerine ek bir sarma katmanı sağlamaz (gereksiz soyutlamaları önlemek).

Örneğin, rasgele bir şifreyi hashlemek için `bcrypt` kullanalım.

İlk olarak gerekli paketleri yükleyin:

```shell
$ npm i bcrypt
$ npm i -D @types/bcrypt
```

Yükleme tamamlandığında, şu şekilde `hash` fonksiyonunu kullanabilirsiniz:

```typescript
import * as bcrypt from 'bcrypt';

const saltOrRounds = 10;
const password = 'rasgele_şifre';
const hash = await bcrypt.hash(password, saltOrRounds);
```

Bir tuz oluşturmak için `genSalt` fonksiyonunu kullanın:

```typescript
const tuz = await bcrypt.genSalt();
```

Şifreyi karşılaştırmak/kontrol etmek için `compare` fonksiyonunu kullanın:

```typescript
const eşleşiyorMu = await bcrypt.compare(password, hash);
```

Daha fazla bilgi için [buradaki](https://www.npmjs.com/package/bcrypt) mevcut fonksiyonları inceleyebilirsiniz.